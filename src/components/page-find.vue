
<style scoped> 
  /** Auto height & width so we get whole page scrolling */
  .page-layout {
    display: grid; 
    grid-template-columns: 1fr minmax(auto, 800px) 1fr;
    grid-template-rows: auto 1fr;
  }
  .grid-place-header {
    grid-column: 1 / 4;
    grid-row: 1 / 2;
  }
  .grid-place-content {
    grid-column: 2 / 3;
    grid-row: 2 / 3;
  }

  /* Move to global? */
  .header {
    padding: 0.5em;
  }

  /* Move to global and rename? */
  .content {
    padding: 0px 5px;
    /* We can get internal scrolling by disabling this and setting height & width on page-layout */
    /* overflow-y: auto; */ 
  }

</style>

<template>
  <div class="page-layout">
    
    <div class="grid-place-header header header-coloring">
      <perma-paste-logo></perma-paste-logo>
    </div>

    <div class="grid-place-content content">
  
      <section class="search-input-section">
        <form class="search-input-form">
          <input :disabled="searching" placeholder="Wallet Address" id="walletAddrInput" type=text v-model="walletAddr"/>
          <button :disabled="searching" class="secondary-btn" @click="$router.push(`/find?wallet=${walletAddr}&n=${(Math.random() * 1000000).toString().slice(0, 5)}`)">Search</button>
        </form>
        <form class="search-input-form">
          <input :disabled="searching" placeholder="Block Number" id="blockInput" type=text v-model="blockHeight"/>
          <button :disabled="searching" class="secondary-btn" @click="$router.push(`/find?block=${blockHeight}&n=${(Math.random() * 1000000).toString().slice(0, 5)}`)">Search</button>
        </form>
      </section>

      <section class="search-results-section"> 

        <div v-if="searching" class="ld ll results-text"> 
          Searching <br/>
          <div style="color: coral; font-size: 2rem; margin: 1em;" class="ld ld-ball ld-bounce">
          </div>
        </div>

        <div v-else> 

          <p class="results-text" v-if="searched === ''">
            Search by wallet or by block number
          </p>

          <p class="results-text" v-if="searched !== '' && errors.length == 0 && totalResultCount > 0" >
            Results for {{ searched }} {{ searched === 'wallet' ? walletAddr : blockHeight }}
          </p>

          <p class="results-text" v-if="searched !== '' && errors.length == 0 && totalResultCount === 0">
            No Results
          </p>
          
          <div class="results-box" v-if="publicPastes.length > 0"> 
            <p>Found Public Pastes</p>
            <table>
              <tr>
                <th>Block</th><th>TX</th><th>Title</th>
              </tr>
              <tr v-for="(result, index) in publicPastes" :key="index">
                <td> {{ result.status.confirmed.block_height }} </td>
                <td class="tx-td"> <router-link v-bind:to="'/view/' + result.id"> {{ result.id }} </router-link> </td>
                <td> {{ result.title }} </td>
              </tr>
            </table>
          </div>
      
          <div class="results-box" v-if="possibleEncryptedPastes.length > 0">
            <p>Found potential Encrypted Pastes</p>
            <table>
              <tr>
                <th>Block</th><th>TX</th>
              </tr>
              <tr v-for="(result, index) in possibleEncryptedPastes" :key="index">
                <td> {{ result.status.confirmed.block_height }} </td>
                <td class="tx-td"> <router-link v-bind:to="'/view/' + result.id"> {{ result.id }} </router-link> </td>
              </tr>
            </table>
          </div>

          <p class="results-errors" v-if="errors.length > 0">
            {{ errors[0] }}
          </p>
        </div>

      </section>

    </div>

  </div>
</template>

<script lang="ts">

import Vue from 'vue'
import Component from 'vue-class-component'
import { Watch } from 'vue-property-decorator'
import { Paste, TYPE_TAG, TYPE_TAG_PUBLIC, TITLE_TAG } from '../lib/pastes'
import { arweave, isValidWalletAddr, ArqlExtraResult, txExtra } from '../lib/permaweb'

@Component
export default class extends Vue {

  walletAddr: string = ''
  blockHeight: string = ''
  
  searching = false;
  searched: 'block' | 'wallet' | ''  = ''

  allTxs: ArqlExtraResult[] = [] 
  publicPastes: (ArqlExtraResult & { title?: string })[] = []

  errors: string[] = [];

  // This (and the @click handlers) is a hack to have search results linkable.
  // The N query parameter is just a random number to always
  // trigger a route change so we can retry on the same block/wallet
  @Watch('$route')
  routeChanged() {
    const query = (this as any).$route.query;
    if (query.wallet) {
      this.walletAddr = query.wallet
      this.blockHeight = ''
      this.searchByWallet()
    }
    if (query.block) {
      this.blockHeight = query.block; 
      this.walletAddr = ''
      this.searchByBlock()
    }
  }

  created() {
    this.routeChanged();
  }

  async searchByWallet() {
    this.searching = true;
    this.searched = 'wallet'
    this.allTxs = []
    this.publicPastes = []
    this.errors = []
    const val = this.walletAddr.trim()
    try {
      this.allTxs = await txExtra(await arweave.arql({ op: "equals", expr1: "from", expr2: val }))
      this.publicPastes = this.allTxs.filter(tx => tx.tags.find(t => t.name === TYPE_TAG && t.value === TYPE_TAG_PUBLIC))
      this.fillTitles()
    }
    catch (e) {
      console.error(e)
      this.errors.push(e.message || e.text || e.statusText || e.status)
    }
    this.searching = false;
  }

  async searchByBlock() {
    this.searching = true
    this.searched = 'block'
    this.allTxs = []
    this.publicPastes = []
    this.errors = []

    const val = Number(this.blockHeight)

    if (Number.isNaN(val) || val === 0) {
      this.errors.push('Invalid Block Number')
      this.searching = false;
      return
    }

    try {
      const block = await arweave.api.get(`/block/height/${val}`).then(x => x.data)
      if (typeof block === 'string') {
        this.errors.push(block) // error (block not found)
      }
      else if (!block.txs) {
        this.errors.push('Invalid block')
      }
      else {
        // Get extra metadata and extract out public pastes.
        this.allTxs = await txExtra(block.txs)
        this.publicPastes = this.allTxs.filter(tx => tx.tags.find(t => t.name === TYPE_TAG && t.value === TYPE_TAG_PUBLIC))
        this.fillTitles()
      }
    }
    catch (e) {
      console.error(e)
      this.errors.push(e.message || e.text || e.statusText || e.status)
    }
    this.searching = false;
  }

  fillTitles() {
    this.publicPastes.forEach(pp => { 
      pp.title = ( 
        pp.tags.filter(x => x.name === TITLE_TAG).map(x => x.value)[0] 
        || 
        'Untitled'
      )
    })
  }

  // Vue computed properties, will update whenever allTxs changes.
  get possibleEncryptedPastes(): ArqlExtraResult[] {
    return this.allTxs.filter(
      tx => tx.tags.length == 0
    )
  }

  get totalResultCount(): number {
    return this.possibleEncryptedPastes.length + this.publicPastes.length
  }
}

</script>

<style scoped>

/* Internal content styles */

.search-input-section {
  padding-top: 1rem;
}
.search-input-form {
  display: flex;
  align-items: stretch;
  justify-items: stretch;
}
.search-input-form input {
  flex-grow: 1;
}
.search-input-form button {
  margin-left: 0.15em;
  padding: 0.5em;
}

.results-text {
  margin-top: 2em;
  text-align: center;
  font-size: 0.8em;
  color: rgb(70, 70, 70)
}
.results-errors {
  text-align: center;
  font-size: 0.85em;
  color: rgb(233, 1, 1);
  padding: 0.7em;
  line-height: 1.6em;
  border: 1px dashed rgb(243, 100, 100, 1);
  border-radius: 1rem;
}
.results-box {
  font-size: 0.9em;
  margin-top: 1.5rem;
  text-align: left;
  padding: 0.3em;
  margin-top: 1.5em;
}
.results-box p {
  margin: 0;
  margin-bottom: 1rem;
}
table {
  background: white;
  width: 100%;
  font-size: 0.9em;
  border-collapse: collapse;
}
th, td {
  border-bottom: 1px solid #ddd;
  text-align: left;
}

.tx-td { 
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 60px;
}  

th {
  color: rgb(30, 30, 30);
}

</style>