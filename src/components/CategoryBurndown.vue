<template>
  <div class="">
    <div class="row">
      <div class="col-xs-12 ml-sm-auto col-lg-12 pt-3 px-4">
        <h1>Category Burndown Chart</h1>
        <span>Track how much you have left in your budget for a category in a month and how quickly you are spending it. This helps you gauge if you are on track to meet your budget goals or if you should slow down your spending in a category.</span>
      </div>
    </div>
    <div class="row">
      <div class="col-xs-6 col-lg-4 pt-3 px-4">
          <div class="d-flex flex-wrap flex-md-nowrap pb-2 mb-3">
            <select class="form-control" v-model="selectedCategory" v-on:change="getTransactionsForSelectedCategory(selectedCategory)">
              <option value="" disabled selected>Choose a category</option>
              <option v-for="category in categories" :value="category">
                {{category.name}}
              </option>
            </select>
          </div>
      </div>
    </div>
    <!-- <div class="row">
        Track how much you have left in your budget for a category in a month and how quickly you are spending it. This helps you gauge if you are on track to meet your budget goals or if you should slow down your spending in a category.
    </div> -->
    <div class="row" v-if="selectedCategory != ''">
      <div class="col-xs-12 ml-sm-auto col-lg-12 pt-3 px-4">
        <div class="d-flex justify-content-between flex-wrap flex-md-nowrap align-items-center pb-2 mb-3 border-bottom">
          <h1 class="h2">{{selectedMonth}}</h1>
          <div class="mb-2 mb-md-0">
              <div class="btn-group mr-2">
                <button class="btn btn-outline-secondary" type="button" @click="goToPreviousMonth()">Previous Month</button>
                <button class="btn btn-outline-secondary" v-if="selectedMonthIndex < 0" type="button" @click="goToNextMonth()">Next Month</button>
              </div>
          </div>
        </div>
      </div>
      <div class="col-xs-12 ml-sm-auto col-lg-12 pt-3 px-4">
          <div id="chart"></div>
          <Transactions :transactions="transactions" />
      </div>
    </div>
  </div>
</template>

<script>
import {utils} from 'ynab';
import Transactions from './Transactions.vue';

export default {
  props: ['categories', 'budgetId', 'api', 'allTransactions'],
  data () {
    return {
      selectedMonthIndex: 0,
      selectedMonth: '',
      selectedCategory: '',
      startingBalance: 0,
      transactions: [],
      idealBurndown: ['ideal'],
      actualBurndown: ['actual'],
      idealX: ['idealX', 1, 2, 3, 4, 5, 6],
      actualX: ['actualX', 1,2, 3, 4]
    }
  },
  methods: {
    refreshChartData(category, transactions, startDate, endDate) {
      this.idealBurndown = ['ideal'];
      this.idealX = ['idealX'];
      this.actualBurndown = ['actual'];
      this.actualX = ['actualX'];
      // var startingBalance = category.balance - category.activity
      var actualEndDate = this.selectedMonthIndex == 0 ? new Date() : endDate
      this.buildActualBurndownData(this.startingBalance, transactions, startDate, actualEndDate);
      this.buildIdealBurndownData(this.startingBalance, startDate, endDate);
      this.buildChart();
    },
    getStartingBalance(category, startDate, endDate, categoryTransactions) {
      this.loading = true
      this.error = null
      this.api.months.getBudgetMonth(this.budgetId, startDate).then((res) => {
        var categoryMonth = res.data.month.categories.find(cat => cat.id === category.id);
        this.startingBalance = categoryMonth.balance - categoryMonth.activity
        this.refreshChartData(this.selectedCategory, categoryTransactions, startDate, endDate);
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    getTransactionsForSelectedCategory() {
      this.loading = true;
      this.error = null;
      this.transactions = [];
      this.selectedMonth = this.getDateName(this.selectedMonthIndex);
      var startDate = this.getFirstDayOfMonth(this.selectedMonthIndex);
      var endDate = this.getLastDayOfMonth(this.selectedMonthIndex);
      var categoryTransactions = [];
      this.api.transactions.getTransactionsByCategory(this.budgetId, this.selectedCategory.id, startDate).then((res) => {
        categoryTransactions = this.parseCategoryTransactions(res.data.transactions, endDate);
        this.transactions = JSON.parse(JSON.stringify(categoryTransactions)).reverse(); //immutable list used to display transactions below the chart
        this.getStartingBalance(this.selectedCategory, startDate, endDate, categoryTransactions)
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    getFirstDayOfMonth(index) {
      var date = new Date();
      return new Date(date.getFullYear(), date.getMonth() + index, 1);
    },
    getLastDayOfMonth(index) {
      var date = new Date();
      return new Date(date.getFullYear(), date.getMonth() + 1 + index, 0 + 1);
    },
    getDateName(index) {
      const monthNames = ["January", "February", "March", "April", "May", "June",
        "July", "August", "September", "October", "November", "December"
      ];

      var date = new Date();
      var monthIndex = index%12;
      var currentMonth = date.getMonth();
      var yearIndex = (index + currentMonth)/12;
      if (monthIndex < -currentMonth) {
        monthIndex += 12
      }
      var year = date.getFullYear() + Math.floor(yearIndex);
      var month = date.getMonth() + monthIndex;
      return `${monthNames[month]} ${year}`;
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

          this.actualBurndown.push(this.convertMilliUnitsToCurrencyAmount(balance, 2))
          this.actualX.push(i + 1)
      }
    },
    buildIdealBurndownData(startingBalance, startDate, endDate) {
      var range = this.dateDiff(startDate, endDate)
      var balance = startingBalance
      var idealDailySpend = startingBalance / range
      idealDailySpend += idealDailySpend / range
      for (var i = 0; i < range; i++) {
          this.idealBurndown.push(this.convertMilliUnitsToCurrencyAmount(balance, 2))
          this.idealX.push(i + 1)

          balance = balance - idealDailySpend
          if (i === range - 2) {
            balance = 0
          }
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
  goToPreviousMonth() {
    this.selectedMonthIndex = this.selectedMonthIndex - 1;
    this.getTransactionsForSelectedCategory();
  },
  goToNextMonth() {
    this.selectedMonthIndex = this.selectedMonthIndex + 1;
    this.getTransactionsForSelectedCategory();
  },
  filterTransactionsByDate(transactions, endDate) {
    return transactions.filter(transaction => {
      if (this.parseDate(transaction.date) < endDate) {
        return transaction;
      }
    });
  },
  parseCategoryTransactions(categoryTransactions, endDate) {
    categoryTransactions = this.filterTransactionsByDate(categoryTransactions, endDate);
    categoryTransactions.map(transaction => {
      if (transaction.parent_transaction_id) {
        var parentTransaction = this.allTransactions.find(t => t.id === transaction.parent_transaction_id);
        if (parentTransaction) {
          transaction.payee_name = parentTransaction.payee_name;
          transaction.category_name = `${transaction.category_name

          } (Split ${parentTransaction.subtransactions.length} categories)`;
        }
      }
    });
    return categoryTransactions;
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
