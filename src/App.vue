<script setup lang="ts">
	import { ref, computed, onMounted } from 'vue'
	import { supabase } from './supabase'
	import './assets/main.css'

	// --- Theme Toggle ---
	const isDark = ref(localStorage.getItem('theme') !== 'light') // Default dark
	const toggleTheme = () => {
	isDark.value = !isDark.value
	localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
	}

	// --- Dynamic Theme Classes ---
	const bgApp = computed(() => isDark.value ? 'bg-slate-950 bg-[radial-gradient(ellipse_80%_80%_at_50%_-20%,rgba(120,119,198,0.3),rgba(255,255,255,0))] text-slate-200' : 'bg-gradient-to-br from-indigo-100 via-purple-50 to-emerald-50 text-slate-800')
	const bgCard = computed(() => isDark.value ? 'bg-white/5 border-white/10 shadow-2xl' : 'bg-white/60 border-white shadow-xl shadow-indigo-900/5')
	const bgInput = computed(() => isDark.value ? 'bg-white/5 border-white/10 focus:bg-white/10 focus:ring-indigo-500/50 text-white placeholder:text-slate-500' : 'bg-white/50 border-white/60 focus:bg-white/80 focus:ring-indigo-100 text-slate-800')
	const bgItem = computed(() => isDark.value ? 'bg-white/5 border-white/10' : 'bg-white/40 border-white/50')
	const btnPrimary = computed(() => isDark.value ? 'bg-indigo-500/80 text-white hover:bg-indigo-500 shadow-[0_0_20px_rgba(99,102,241,0.3)]' : 'bg-slate-900/90 text-white hover:bg-slate-800 shadow-lg shadow-slate-900/20')
	const btnSecondary = computed(() => isDark.value ? 'bg-white/5 border-white/10 hover:bg-white/10 text-slate-300' : 'bg-white/60 border-white hover:bg-white/80 text-slate-600')
	const textTitle = computed(() => isDark.value ? 'text-white' : 'bg-gradient-to-r from-indigo-800 to-purple-600 bg-clip-text text-transparent')
	const textSub = computed(() => isDark.value ? 'text-slate-400' : 'text-slate-500')
	const optionBg = computed(() => isDark.value ? 'bg-slate-900 text-white' : 'bg-white text-slate-800')

	const netPay = ref(Number(localStorage.getItem('netPay')) || 0)
	const hasStarted = ref(!!localStorage.getItem('netPay'))

	const inputNetPay = ref(netPay.value || '')
	const fixedBills = ref<{ name: string; amount: number | number }[]>([])
	const newBill = ref({ name: '', amount: 0 })

	interface Transaction {
		id: string;
		created_at: string;
		amount: number;
		category: string;
		description: string;
	}

	const transactions = ref<Transaction[]>([])
	const newTxAmount = ref('')
	const newTxCategory = ref('')
	const newTxDesc = ref('')
	const isResetting = ref(false)

	const fetchTransactions = async () => {
		const { data, error } = await supabase.from('transactions').select('*').order('created_at', { ascending: false })

		if (data && !error) transactions.value = data
	}

	onMounted(() => {
		if (hasStarted.value) fetchTransactions()
	})

	const addFixedBill = () => {
		if (!newBill.value.name || !newBill.value.amount) return
		fixedBills.value.push({ ...newBill.value })
		newBill.value = { name: '', amount: 0 }
	}

	const removeFixedBill = (index: number) => {
		fixedBills.value.splice(index, 1)
	}

	const startTracking = async () => {
		if (!inputNetPay.value) return
		netPay.value = Number(inputNetPay.value)
		localStorage.setItem('netPay', netPay.value.toString())
		hasStarted.value = true

		if (fixedBills.value.length > 0) {
			const billsToInsert = fixedBills.value.map((bill) => ({
				amount: Number(bill.amount),
				category: 'Needs',
				description: bill.name,
			}))

			await supabase.from('transactions').insert(billsToInsert)
		}
		await fetchTransactions()
	}

	const addTransaction = async () => {
	if (!newTxAmount.value || !newTxDesc.value) return
	
	const { data, error } = await supabase
		.from('transactions')
		.insert([{
		amount: Number(newTxAmount.value),
		category: newTxCategory.value,
		description: newTxDesc.value
		}])
		.select()

		if (data && !error) {
			transactions.value.unshift(data[0])
			newTxAmount.value = ''
			newTxDesc.value = ''
		}
	}

	const deleteTransaction = async (id: string) => {
		const { error } = await supabase.from('transactions').delete().eq('id', id)
		if (!error) {
			transactions.value = transactions.value.filter(t => t.id !== id)
		}
	}

	const handleReset = async () => {
		if (!isResetting.value) {
			isResetting.value = true
			setTimeout(() => { isResetting.value = false }, 3000)
			return
		}
		
		// Confirmed reset: Clear database and local storage
		await supabase.from('transactions').delete().not('id', 'is', null)
		localStorage.removeItem('netPay')
		netPay.value = 0
		hasStarted.value = false
		transactions.value = []
		fixedBills.value = []
		isResetting.value = false
	}
	const totalNeeds = computed(() => transactions.value.filter(t => t.category === 'Needs').reduce((sum, t) => sum + Number(t.amount), 0))
	const totalWants = computed(() => transactions.value.filter(t => t.category === 'Wants').reduce((sum, t) => sum + Number(t.amount), 0))
	const totalSavings = computed(() => transactions.value.filter(t => t.category === 'Savings').reduce((sum, t) => sum + Number(t.amount), 0))

	const formatCurrency = (val: number) => new Intl.NumberFormat('en-PH', { style: 'currency', currency: 'PHP' }).format(val)

</script>

<template>
  <div class="min-h-screen overflow-x-hidden transition-colors duration-500 font-sans p-4 md:p-8" :class="bgApp">
    <div class="max-w-xl mx-auto">
      
      <!-- ONBOARDING VIEW -->
      <div v-if="!hasStarted" class="backdrop-blur-xl p-8 rounded-3xl border transition-all" :class="bgCard">
        <div class="flex justify-between items-start mb-2">
          <h1 class="text-3xl font-bold" :class="textTitle">Setup Your Tracker</h1>
          <button @click="toggleTheme" class="p-2 backdrop-blur-md border rounded-xl flex items-center justify-center transition-all w-10 h-10" :class="btnSecondary" title="Toggle Theme">
            <span v-if="isDark">☀️</span><span v-else>🌙</span>
          </button>
        </div>
        <p class="mb-8 font-medium" :class="textSub">Enter your take-home pay and log your recurring bills.</p>
        
        <div class="mb-6">
          <label class="block text-sm font-bold mb-2" :class="textSub">Monthly Net Pay (PHP)</label>
          <input v-model="inputNetPay" type="number" placeholder="e.g. 44592" class="w-full p-4 rounded-2xl outline-none transition-all" :class="bgInput">
        </div>

        <div class="mb-8">
          <label class="block text-sm font-bold mb-2" :class="textSub">Fixed Bills</label>
          <div class="flex flex-col sm:flex-row gap-3 mb-4">
			<input v-model="newBill.name" type="text" placeholder="Bill Name" class="flex-1 p-4 rounded-2xl outline-none transition-all" :class="bgInput">
			<div class="flex gap-3">
				<input v-model="newBill.amount" type="number" placeholder="Amount" class="flex-1 sm:w-32 p-4 rounded-2xl outline-none transition-all" :class="bgInput">
				<button @click="addFixedBill" class="px-6 font-bold rounded-2xl transition-colors border shadow-sm" :class="btnSecondary">+</button>
			</div>
		</div>
          
          <div v-for="(bill, index) in fixedBills" :key="index" class="flex justify-between items-center p-4 rounded-2xl mb-2 border shadow-sm" :class="bgItem">
            <span class="font-semibold">{{ bill.name }}</span>
            <div class="flex items-center gap-4">
              <span class="font-bold">{{ formatCurrency(Number(bill.amount)) }}</span>
              <button @click="removeFixedBill(index)" class="text-red-400 hover:text-red-500 font-bold">✕</button>
            </div>
          </div>
        </div>

        <button @click="startTracking" :disabled="!inputNetPay" class="w-full py-4 font-bold rounded-2xl disabled:opacity-50 transition-all" :class="btnPrimary">
          Start Tracking
        </button>
      </div>

      <!-- DASHBOARD VIEW -->
      <div v-else>
        <!-- Header -->
        <div class="flex justify-between items-end mb-8">
          <div>
            <h1 class="text-3xl font-bold" :class="textTitle">Overview</h1>
            <p class="font-medium mt-1" :class="textSub">Based on {{ formatCurrency(netPay) }} Net Pay</p>
          </div>
          <div class="flex gap-2">
            <button @click="toggleTheme" class="p-2.5 backdrop-blur-md border shadow-sm rounded-xl flex items-center justify-center transition-all w-11 h-11" :class="btnSecondary" title="Toggle Theme">
              <span v-if="isDark">☀️</span><span v-else>🌙</span>
            </button>
            <button @click="handleReset" class="px-5 py-2.5 backdrop-blur-md border shadow-sm rounded-xl text-sm font-semibold transition-all" :class="[btnSecondary, isResetting ? '!text-red-400 !border-red-500/30 !bg-red-500/10' : '']">
              {{ isResetting ? 'Confirm Reset' : 'Reset App' }}
            </button>
          </div>
        </div>

        <!-- Progress Cards -->
        <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
          <div class="backdrop-blur-xl p-6 rounded-3xl border relative overflow-hidden group transition-colors" :class="bgCard">
            <div class="absolute top-0 left-0 w-full h-1" :class="isDark ? 'bg-white/5' : 'bg-indigo-100'"><div class="h-full transition-all duration-1000 ease-out" :class="isDark ? 'bg-indigo-500 shadow-[0_0_10px_rgba(99,102,241,0.8)]' : 'bg-indigo-500'" :style="{ width: Math.min((totalNeeds / (netPay * 0.5)) * 100, 100) + '%' }"></div></div>
            <div class="text-xs font-bold mb-2 uppercase tracking-wider mt-2" :class="isDark ? 'text-indigo-400' : 'text-indigo-500'">Needs (50%)</div>
            <div class="text-2xl font-bold mb-1" :class="isDark ? 'text-white' : 'text-slate-800'">{{ formatCurrency(totalNeeds) }}</div>
            <div class="text-xs font-medium" :class="textSub">Limit: {{ formatCurrency(netPay * 0.5) }}</div>
          </div>

          <div class="backdrop-blur-xl p-6 rounded-3xl border relative overflow-hidden group transition-colors" :class="bgCard">
            <div class="absolute top-0 left-0 w-full h-1" :class="isDark ? 'bg-white/5' : 'bg-purple-100'"><div class="h-full transition-all duration-1000 ease-out" :class="isDark ? 'bg-purple-500 shadow-[0_0_10px_rgba(168,85,247,0.8)]' : 'bg-purple-500'" :style="{ width: Math.min((totalWants / (netPay * 0.3)) * 100, 100) + '%' }"></div></div>
            <div class="text-xs font-bold mb-2 uppercase tracking-wider mt-2" :class="isDark ? 'text-purple-400' : 'text-purple-500'">Wants (30%)</div>
            <div class="text-2xl font-bold mb-1" :class="isDark ? 'text-white' : 'text-slate-800'">{{ formatCurrency(totalWants) }}</div>
            <div class="text-xs font-medium" :class="textSub">Limit: {{ formatCurrency(netPay * 0.3) }}</div>
          </div>

          <div class="backdrop-blur-xl p-6 rounded-3xl border relative overflow-hidden group transition-colors" :class="bgCard">
            <div class="absolute top-0 left-0 w-full h-1" :class="isDark ? 'bg-white/5' : 'bg-emerald-100'"><div class="h-full transition-all duration-1000 ease-out" :class="isDark ? 'bg-emerald-500 shadow-[0_0_10px_rgba(16,185,129,0.8)]' : 'bg-emerald-500'" :style="{ width: Math.min((totalSavings / (netPay * 0.2)) * 100, 100) + '%' }"></div></div>
            <div class="text-xs font-bold mb-2 uppercase tracking-wider mt-2" :class="isDark ? 'text-emerald-400' : 'text-emerald-500'">Savings (20%)</div>
            <div class="text-2xl font-bold mb-1" :class="isDark ? 'text-white' : 'text-slate-800'">{{ formatCurrency(totalSavings) }}</div>
            <div class="text-xs font-medium" :class="textSub">Target: {{ formatCurrency(netPay * 0.2) }}</div>
          </div>
        </div>

        <!-- Add Transaction Form (Flex-Wrap Fix) -->
        <div class="backdrop-blur-xl p-4 rounded-3xl border mb-8 flex flex-wrap gap-3" :class="bgCard">
          <input v-model="newTxDesc" type="text" placeholder="What did you buy?" class="flex-[1_1_250px] p-4 rounded-2xl outline-none transition-all" :class="bgInput">
          <input v-model="newTxAmount" type="number" placeholder="₱ Amount" class="flex-[1_1_120px] p-4 rounded-2xl outline-none transition-all" :class="bgInput">
          <select v-model="newTxCategory" class="flex-[1_1_120px] p-4 rounded-2xl outline-none transition-all font-semibold" :class="bgInput">
            <option :class="optionBg">Needs</option>
            <option :class="optionBg">Wants</option>
            <option :class="optionBg">Savings</option>
          </select>
          <button @click="addTransaction" class="flex-[1_1_100px] px-8 font-bold rounded-2xl transition-colors" :class="btnPrimary">Add</button>
        </div>

        <!-- Recent Activity -->
        <h2 class="text-lg font-bold mb-4 ml-2" :class="textSub">Recent Transactions</h2>
        <div class="flex flex-col gap-3">
          <div v-for="tx in transactions" :key="tx.id" class="flex justify-between items-center p-5 backdrop-blur-md rounded-3xl border transition-all group" :class="bgItem">
            <div class="flex flex-col">
              <span class="font-bold text-lg" :class="isDark ? 'text-white' : 'text-slate-800'">{{ tx.description }}</span>
              <span class="text-xs font-bold tracking-wide mt-1" 
                :class="{'text-indigo-400': tx.category === 'Needs', 'text-purple-400': tx.category === 'Wants', 'text-emerald-400': tx.category === 'Savings'}">
                {{ tx.category }}
              </span>
            </div>
            <div class="flex items-center gap-5">
              <span class="font-bold text-xl" :class="isDark ? 'text-slate-200' : 'text-slate-700'">{{ formatCurrency(tx.amount) }}</span>
              <button @click="deleteTransaction(tx.id)" class="w-10 h-10 flex items-center justify-center rounded-xl transition-all border shadow-sm" :class="isDark ? 'bg-white/5 text-slate-500 hover:bg-red-500/20 hover:text-red-400 border-white/5' : 'bg-white/80 text-slate-400 hover:bg-red-50 hover:text-red-500 border-white'" title="Delete">
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"></path><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path></svg>
              </button>
            </div>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>

<style scoped></style>
