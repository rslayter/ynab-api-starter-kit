<template>
  <div class="categoryBurndown">
    <div class="row">
      <h1>Category Burndown Chart</h1>
    </div>
    <div id="selectCategoryDropdown" class="row">
      <div class="col-xs-4">
        <select class="form-control" v-model="selectedCategory" v-on:change="selectCategory(selectedCategory)">
          <option value="" disabled selected>Choose a category</option>
          <option v-for="category in categories" :value="category">
            {{category.name}}
          </option>
        </select>
      </div>
    </div>
    <div class="row">
        Track how much you have left in your budget for a category in a month and how quickly you are spending it. This helps you gauge if you are on track to meet your budget goals or if you should slow down your spending in a category. 
    </div>
    <div class="row" v-if="selectedCategory != ''">
      <div class="col-xs-4">
          <div id="chart"></div>
      </div>
      <Transactions :transactions="transactions" />
    </div>
  </div>
</template>

<script>
import {utils} from 'ynab';
import Transactions from './Transactions.vue';

export default {
  props: ['categories', 'budgetId', 'api'],
  data () {
    return {
      selectedCategory: '',
      transactions: [],
      idealBurndown: ['ideal'],
      actualBurndown: ['actual'],
      idealX: ['idealX', 1, 2, 3, 4, 5, 6],
      actualX: ['actualX', 1,2, 3, 4]
    }
  },
  methods: {
    selectCategory(category) {
      //TODO generate idealBurndown
      //TODO get actual burndown
      //TODO populate x and y axis
      this.getTransactionsForSelectedCategory()
    },
    refreshChartData(category, transactions, startDate, endDate) {
      this.idealBurndown = ['ideal'];
      this.idealX = ['idealX'];
      this.actualBurndown = ['actual'];
      this.actualX = ['actualX'];
      var startingBalance = category.balance - category.activity
      this.buildActualBurndownData(startingBalance, transactions, startDate, new Date());
      this.buildIdealBurndownData(startingBalance, startDate, endDate);
      this.buildChart();
    },
    getTransactionsForSelectedCategory() {
      this.loading = true;
      this.error = null;
      this.transactions = [];
      this.api.transactions.getTransactionsByCategory(this.budgetId, this.selectedCategory.id, this.getFirstDayOfMonth()).then((res) => {
        this.transactions = JSON.parse(JSON.stringify(res.data.transactions)).reverse();
        this.refreshChartData(this.selectedCategory, res.data.transactions, this.getFirstDayOfMonth(), this.getLastDayOfMonth());
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    getFirstDayOfMonth() {
      var date = new Date();
      return new Date(date.getFullYear(), date.getMonth(), 1);
    },
    getLastDayOfMonth() {
      var date = new Date();
      return new Date(date.getFullYear(), date.getMonth() + 1, 0);
    },
    dateDiff(first, second) {
      // Take the difference between the dates and divide by milliseconds per day.
      // Round to nearest whole number to deal with DST.
      return Math.round((second-first)/(1000*60*60*24));
    },
    addDays(date, days) {
      var result = new Date(date);
      result.setDate(result.getDate() + days);
      return result;
    },
    parseDate(dateString) {
      var parts = dateString.split('-');
      // Please pay attention to the month (parts[1]); JavaScript counts months from 0:
      // January - 0, February - 1, etc.
      return new Date(parts[0], parts[1] - 1, parts[2]);
    },
    buildActualBurndownData(startingBalance, transactions, startDate, endDate) {
      var range = this.dateDiff(startDate, endDate)
      var balance = startingBalance
      var remainingTransactions = transactions
      for (var i = 0; i < range; i++) {
          var dataDate = this.addDays(startDate, i)
          while (remainingTransactions.length > 0) {
            var transactionDate = this.parseDate(remainingTransactions[0].date)
            if (dataDate.getTime() === transactionDate.getTime()) {
              balance += remainingTransactions[0].amount
              remainingTransactions.shift()
            } else {
              break
            }
          }

          this.actualBurndown.push(this.convertMilliUnitsToCurrencyAmount(balance).toFixed(2))
          this.actualX.push(i + 1)
      }
    },
    buildIdealBurndownData(startingBalance, startDate, endDate) {
      var range = this.dateDiff(startDate, endDate)
      var balance = startingBalance
      var idealDailySpend = startingBalance / range
      for (var i = 0; i < range; i++) {
          this.idealBurndown.push(this.convertMilliUnitsToCurrencyAmount(balance).toFixed(2))
          this.idealX.push(i + 1)

          balance = balance - idealDailySpend
      }
    },
    buildChart() {
      var chart = c3.generate({
        data: {
        xs: {
            'ideal': 'idealX',
            'actual': 'actualX',
        },
        columns: [
            this.idealX, this.actualX, this.idealBurndown, this.actualBurndown
          ]
        },
        axis: {
          x: {
            label: {
              text: 'Day of Month',
              position: 'outer-middle'
            }
          },
          y: {
            // min: Math.min(this.selectedCategory.balance, 0),
            padding: {bottom: 0},
            label: {
              text: 'Remaining Balance',
              position: 'outer-middle'
            },
            tick: {
              format: d3.format("$,")
            }
          }
        },
        legend: {
          position: 'bottom'
        }
      });
  },
  // Now we can make this method available to our template
  // So we can format this milliunits in the correct currency format
  convertMilliUnitsToCurrencyAmount: utils.convertMilliUnitsToCurrencyAmount
},
// Specify which components we want to make available to our templates
components: {
  Transactions
}
}
</script>
