<template>
  <div id="app">
    <Nav />
    <div class="container">
      <div class="row">
        <div class="col-xs-12 ml-sm-auto col-lg-12 pt-3 px-4">
          <h1>Category Tracker</h1>
          <h3 v-if="loading" class="display-4">Loading...</h3>
          <div v-else>
            <div v-if="error">
              <h3 class="display-4">Oops!</h3>
              <p class="lead">{{error}}</p>
              <button class="btn btn-primary" @click="resetToken">Try Again &gt;</button>
            </div>
            <div v-else>
              <form v-if="!ynab.token">
                <div class="form-group">
                  <h3 class="display-5">Hello!</h3>
                  <p class="lead">If you would like to use this App, please authorize with YNAB!</p>
                  <button @click="authorizeWithYNAB" class="btn btn-primary">Authorize This App With YNAB &gt;</button>
                </div>
              </form>
              <Budgets v-else-if="!budgetId" :budgets="budgets" :selectBudget="selectBudget" />
              <div v-else>
                <!-- Show the burndown chart for the selected budget -->
                <CategoryBurndown :budgetId="budgetId" :categories="categories" :api="api" :allTransactions="allTransactions"/>
                <div class="row">
                  <div class="col-xs-12 ml-sm-auto col-lg-12 pt-3 px-4">
                    <button class="btn btn-info back-button" @click="budgetId = null">&lt; Select Another Budget</button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <Footer />
    </div>
  </div>
</template>

<script>
import * as ynab from 'ynab';
import config from './config.json';
import Nav from './components/Nav.vue';
import Footer from './components/Footer.vue';
import Budgets from './components/Budgets.vue';
import CategoryBurndown from './components/CategoryBurndown.vue';

export default {
  data () {
    return {
      ynab: {
        clientId: config.clientId,
        redirectUri: config.redirectUri,
        token: null,
        api: null,
      },
      loading: false,
      error: null,
      budgetId: null,
      budgets: [],
      categories: [],
      allTransactions: []
    }
  },
  // When this component is created, check whether we need to get a token,
  // budgets or display the transactions
  created() {
    this.ynab.token = this.findYNABToken();
    if (this.ynab.token) {
      this.api = new ynab.api(this.ynab.token);
      if (!this.budgetId) {
        this.getBudgets();
      } else {
        this.selectBudget(this.budgetId);
      }
    }
  },
  methods: {
    // This uses the YNAB API to get a list of budgets
    getBudgets() {
      this.loading = true;
      this.error = null;
      this.api.budgets.getBudgets().then((res) => {
        this.budgets = res.data.budgets;
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    // This selects a budget and gets all the categories and transactions in that budget
    selectBudget(id) {
      this.loading = true;
      this.error = null;
      this.budgetId = id;
      this.categories = [];
      this.api.categories.getCategories(id).then((res) => {
        this.categories = res.data.category_groups
        .filter(category => {
          return category.name != "Internal Master Category" && category.name != "Credit Card Payments" && category.name != "Hidden Categories"
        })
        .reduce((categoryAcc, category_group) => {
          return categoryAcc.concat(category_group.categories);
        }, []);
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });

      this.allTransactions = [];
      this.api.transactions.getTransactions(id).then((res) => {
        this.allTransactions = res.data.transactions;
      }).catch((err) => {
        this.error = err.error.detail;
      });
    },
    // This builds a URI to get an access token from YNAB
    // https://api.youneedabudget.com/#outh-applications
    authorizeWithYNAB(e) {
      e.preventDefault();
      const uri = `https://app.youneedabudget.com/oauth/authorize?client_id=${this.ynab.clientId}&redirect_uri=${this.ynab.redirectUri}&response_type=token`;
      location.replace(uri);
    },
    // Method to find a YNAB token
    // First it looks in the location.hash and then sessionStorage
    findYNABToken() {
      let token = null;
      const search = window.location.hash.substring(1).replace(/&/g, '","').replace(/=/g,'":"');
      if (search && search !== '') {
        // Try to get access_token from the hash returned by OAuth
        const params = JSON.parse('{"' + search + '"}', function(key, value) {
          return key === '' ? value : decodeURIComponent(value);
        });
        token = params.access_token;
        sessionStorage.setItem('ynab_access_token', token);
        window.location.hash = '';
      } else {
        // Otherwise try sessionStorage
        token = sessionStorage.getItem('ynab_access_token');
      }
      return token;
    },
    // Clear the token and start authorization over
    resetToken() {
      sessionStorage.removeItem('ynab_access_token');
      this.ynab.token = null;
      this.error = null;
    }
  },
  // Specify which components we want to make available to our templates
  components: {
    Nav,
    Footer,
    Budgets,
    CategoryBurndown
  }
}
</script>
