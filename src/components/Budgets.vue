<template>
  <div class="budgets">
    <p>A comprehensive budget should have many different types of categories:</p>
    <ul>
      <li>categories that you spend consistently throughout the month (food, gas, spending money)
      <li>categories that accrue to a goal (vacation, savings, maintenance, etc.)</li>
      <li>categories that have a set value with the same transactions every month (rent, bills)</li>
      <li>miscellaneous categories</li>
    </ul>
    <p>This tool is for categories that you spend throughout the month. Not all categories will make sense for this. For example, this tool will not be very useful for a “Rent” category, but is extremely useful for a category like “Fast Food” or “Spending Money”.</p>
    <p>You can quickly look and see if you are on track for the month or if you should slow down your spending in that category.</p>
    <p>For those developers out there, this is a burndown chart for your categories.</p>
    <h5>Select a budget to get started</h5>
    <div v-for="budget in budgets">
      <a href="#" @click="selectBudget(budget.id, $event)">
        {{budget.name}}
      </a>
    </div>
  </div>
</template>

<script>
export default {
  props: ['budgets', 'selectBudget']
}
</script>
