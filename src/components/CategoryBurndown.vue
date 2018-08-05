<template>
  <div class="categoryBurndown container">
    <h5>Select A Category</h5>
    <div class="row">
      <div class="col-xs-4">
        <select class="form-control" v-model="selectedCategory" v-on:change="selectCategory(selectedCategory)">
          <option v-for="category in categories" :value="category">
            {{category.name}}
          </option>
        </select>
      </div>
    </div>
    <div class="row" v-if="selectedCategory != ''">
      <div class="col-xs-4">
          <div id="chart"></div>
      </div>
    </div>
  </div>
</template>

<script>
import {utils} from 'ynab';

export default {
  props: ['categories', 'budgetId', 'api'],
  data () {
    return {
      selectedCategory: this.categories[0],
      transactions: [],
      idealBurndown: ['ideal', 30, 200, 100, 400, 150, 250],
      actualBurndown: ['actual', 50, 20, 10, 40],
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
        // this.transactions = res.data.transactions;
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
          position: 'right'
        }
      });
  },
  // Now we can make this method available to our template
  // So we can format this milliunits in the correct currency format
  convertMilliUnitsToCurrencyAmount: utils.convertMilliUnitsToCurrencyAmount
},
  // created: function() {
  //     this.selectCategory(this.selectedCategory);
  // }
}
</script>
