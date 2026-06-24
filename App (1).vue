<template>
  <div class="app">

    <!-- HEADER -->
    <header class="header">
      <div class="header-label">⚽ Bolão Oficial ⚽</div>
      <div class="header-teams">
        <span class="flag">🇧🇷</span>
        <div class="header-vs-box">
          <div class="team-name">BRASIL</div>
          <div class="vs-text">vs</div>
          <div class="team-name">ESCÓCIA</div>
        </div>
        <span class="flag">🏴󠁧󠁢󠁳󠁣󠁴󠁿</span>
      </div>
      <div class="header-badge">💰 Entrada: R$ 10,00</div>
      <div class="header-date">24 de Junho de 2026 • 19h (Brasília)</div>
    </header>

    <main class="main">

      <!-- COUNTDOWN -->
      <div v-if="!matchClosed" class="countdown-box">
        <div class="countdown-label">⏳ Palpites encerram em</div>
        <div class="countdown-units">
          <div class="countdown-unit">
            <span class="countdown-value">{{ pad(countdown.d) }}</span>
            <span class="countdown-sub">dias</span>
          </div>
          <div class="countdown-unit">
            <span class="countdown-value">{{ pad(countdown.h) }}</span>
            <span class="countdown-sub">horas</span>
          </div>
          <div class="countdown-unit">
            <span class="countdown-value">{{ pad(countdown.m) }}</span>
            <span class="countdown-sub">min</span>
          </div>
          <div class="countdown-unit">
            <span class="countdown-value">{{ pad(countdown.s) }}</span>
            <span class="countdown-sub">seg</span>
          </div>
        </div>
        <div class="countdown-date">24/06/2026 às 19h — Brasília</div>
      </div>
      <div v-else class="closed-banner">⏰ Palpites encerrados! O jogo já começou.</div>

      <!-- STATS -->
      <div class="stats-card">
        <div>
          <div class="stats-label">Total Arrecadado</div>
          <div class="stats-value green">R$ {{ formatMoney(totalConfirmed) }}</div>
          <div class="stats-sub">{{ confirmedBets.length }} pag. confirmado{{ confirmedBets.length !== 1 ? 's' : '' }}</div>
        </div>
        <div class="stats-divider"></div>
        <div style="text-align:right">
          <div class="stats-label">Participantes</div>
          <div class="stats-value dark">{{ bets.length }}</div>
          <div class="stats-sub">palpite{{ bets.length !== 1 ? 's' : '' }} registrado{{ bets.length !== 1 ? 's' : '' }}</div>
        </div>
      </div>

      <!-- FORM -->
      <div v-if="!matchClosed" class="form-card">
        <h2 class="section-title">📝 Faça seu palpite</h2>

        <div class="field">
          <label class="field-label">Nome Completo</label>
          <input
            v-model="form.name"
            type="text"
            placeholder="Seu nome completo"
            class="field-input"
          />
        </div>

        <div class="field">
          <label class="field-label">Placar do Jogo</label>
          <div class="score-picker">
            <div class="score-team">
              <span class="score-flag">🇧🇷</span>
              <span class="score-team-name green">BRASIL</span>
              <div class="goal-input">
                <button class="goal-btn" @click="decrement('brazil')">−</button>
                <div class="goal-value">{{ form.brazilGoals }}</div>
                <button class="goal-btn" @click="increment('brazil')">+</button>
              </div>
            </div>
            <div class="score-x">×</div>
            <div class="score-team">
              <span class="score-flag">🏴󠁧󠁢󠁳󠁣󠁴󠁿</span>
              <span class="score-team-name dark">ESCÓCIA</span>
              <div class="goal-input">
                <button class="goal-btn" @click="decrement('scotland')">−</button>
                <div class="goal-value">{{ form.scotlandGoals }}</div>
                <button class="goal-btn" @click="increment('scotland')">+</button>
              </div>
            </div>
          </div>

          <div v-if="currentScoreCount > 0" class="score-warning" :class="{ danger: currentScoreCount >= 2 }">
            {{ currentScoreCount >= 2 ? '🚫 Placar lotado! Escolha outro.' : '⚠️ 1 pessoa já escolheu esse placar (máx. 2)' }}
          </div>
        </div>

        <div v-if="formError" class="error-box">{{ formError }}</div>

        <button
          class="submit-btn"
          :disabled="submitting || currentScoreCount >= 2"
          :class="{ disabled: currentScoreCount >= 2 }"
          @click="handleSubmit"
        >
          {{ submitting ? 'Registrando...' : '✅ Confirmar Palpite — R$ 10,00' }}
        </button>
      </div>

      <div v-else class="closed-card">
        <div class="closed-icon">🔒</div>
        <div class="closed-title">Palpites encerrados!</div>
        <div class="closed-sub">O período de apostas foi encerrado.</div>
      </div>

      <!-- BETS LIST -->
      <div class="bets-section">
        <div class="bets-header">
          <h2 class="section-title">🏆 Palpites Registrados</h2>
          <span class="bets-badge">{{ bets.length }}</span>
        </div>

        <div v-if="loading" class="loading">Carregando...</div>

        <div v-else-if="bets.length === 0" class="empty-state">
          Nenhum palpite ainda. Seja o primeiro! 🎯
        </div>

        <div v-else class="bets-list">
          <div
            v-for="bet in [...bets].reverse()"
            :key="bet.id"
            class="bet-card"
            :class="{ confirmed: bet.status === 'confirmed' }"
          >
            <div class="bet-info">
              <div class="bet-name">{{ bet.name }}</div>
              <div class="bet-score">
                🇧🇷 Brasil
                <strong class="green">{{ bet.brazilGoals }}</strong>
                ×
                <strong class="dark">{{ bet.scotlandGoals }}</strong>
                Escócia 🏴󠁧󠁢󠁳󠁣󠁴󠁿
              </div>
            </div>
            <div class="bet-status-col">
              <span class="bet-status" :class="bet.status">
                {{ bet.status === 'confirmed' ? '🟢 Confirmado' : '🟡 Aguardando PIX' }}
              </span>
              <button
                v-if="isAdmin && bet.status === 'pending'"
                class="confirm-btn"
                @click="confirmPayment(bet.id)"
              >
                Confirmar ✓
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- ADMIN -->
      <div class="admin-card">
        <div v-if="!isAdmin">
          <div class="admin-label">🔐 Área do Administrador</div>
          <div class="admin-row">
            <input
              v-model="adminInput"
              type="password"
              placeholder="Senha admin"
              class="field-input"
              @keydown.enter="handleAdminLogin"
            />
            <button class="admin-btn" @click="handleAdminLogin">Entrar</button>
          </div>
          <div v-if="adminError" class="admin-error">{{ adminError }}</div>
        </div>
        <div v-else class="admin-logged">
          <div class="admin-logged-title">✅ Admin logado</div>
          <button class="admin-logout" @click="isAdmin = false">Sair</button>
          <div class="admin-hint">Clique em "Confirmar ✓" para validar pagamentos PIX.</div>
        </div>
      </div>

      <footer class="footer">Desenvolvido por <strong>Zuvinha</strong> ⚽</footer>

    </main>

    <!-- PAYMENT MODAL -->
    <div v-if="pendingBet" class="modal-overlay" @click.self="pendingBet = null">
      <div class="modal">
        <div class="modal-icon">💸</div>
        <h2 class="modal-title">Faça o PIX para confirmar!</h2>
        <p class="modal-sub">Após o pagamento, envie o comprovante.</p>

        <div class="pix-box">
          <div class="pix-value">R$ 10,00</div>
          <div class="pix-label">Chave PIX:</div>
          <div class="pix-key">71 992790879</div>
          <div class="pix-name">Para: Elaine Cerqueira</div>
        </div>

        <div class="modal-tip">📲 Envie o comprovante para confirmar seu palpite.</div>

        <div class="modal-bet-info">
          <div class="modal-bet-title">Seu palpite:</div>
          <div>🇧🇷 Brasil {{ pendingBet.brazilGoals }} × {{ pendingBet.scotlandGoals }} Escócia 🏴󠁧󠁢󠁳󠁣󠁴󠁿</div>
          <div class="modal-bet-name">👤 {{ pendingBet.name }}</div>
        </div>

        <button class="submit-btn" @click="pendingBet = null">Entendido, vou pagar! ✅</button>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { db } from './firebase'
import {
  collection, addDoc, updateDoc,
  doc, onSnapshot, query, orderBy
} from 'firebase/firestore'

// ─── Constantes ───────────────────────────────────────────────
const MATCH_DATE = new Date('2026-06-24T22:00:00Z') // 19h BRT
const MAX_SAME = 2

// ─── Estado ───────────────────────────────────────────────────
const bets = ref([])
const loading = ref(true)
const matchClosed = ref(false)

const form = ref({ name: '', brazilGoals: 0, scotlandGoals: 0 })
const formError = ref('')
const submitting = ref(false)
const pendingBet = ref(null)

const isAdmin = ref(false)
const adminInput = ref('')
const adminError = ref('')

const countdown = ref({ d: 0, h: 0, m: 0, s: 0 })

// ─── Computed ─────────────────────────────────────────────────
const confirmedBets = computed(() => bets.value.filter(b => b.status === 'confirmed'))
const totalConfirmed = computed(() => confirmedBets.value.length * 10)

const scoreMap = computed(() => {
  const map = {}
  bets.value.forEach(b => {
    const k = `${b.brazilGoals}-${b.scotlandGoals}`
    map[k] = (map[k] || 0) + 1
  })
  return map
})

const currentScoreCount = computed(() => {
  const k = `${form.value.brazilGoals}-${form.value.scotlandGoals}`
  return scoreMap.value[k] || 0
})

// ─── Firebase: escuta em tempo real ───────────────────────────
let unsubscribe = null

onMounted(() => {
  matchClosed.value = new Date() >= MATCH_DATE
  startCountdown()

  const q = query(collection(db, 'bets'), orderBy('createdAt', 'asc'))
  unsubscribe = onSnapshot(q, (snapshot) => {
    bets.value = snapshot.docs.map(d => ({ id: d.id, ...d.data() }))
    loading.value = false
  })
})

onUnmounted(() => {
  if (unsubscribe) unsubscribe()
  stopCountdown()
})

// ─── Countdown ────────────────────────────────────────────────
let countdownInterval = null

function startCountdown() {
  function calc() {
    const diff = MATCH_DATE - new Date()
    if (diff <= 0) { matchClosed.value = true; stopCountdown(); return }
    countdown.value = {
      d: Math.floor(diff / 86400000),
      h: Math.floor((diff % 86400000) / 3600000),
      m: Math.floor((diff % 3600000) / 60000),
      s: Math.floor((diff % 60000) / 1000)
    }
  }
  calc()
  countdownInterval = setInterval(calc, 1000)
}

function stopCountdown() {
  if (countdownInterval) clearInterval(countdownInterval)
}

function pad(n) { return String(n).padStart(2, '0') }

// ─── Formulário ───────────────────────────────────────────────
function increment(team) {
  if (team === 'brazil') form.value.brazilGoals = Math.min(20, form.value.brazilGoals + 1)
  else form.value.scotlandGoals = Math.min(20, form.value.scotlandGoals + 1)
}
function decrement(team) {
  if (team === 'brazil') form.value.brazilGoals = Math.max(0, form.value.brazilGoals - 1)
  else form.value.scotlandGoals = Math.max(0, form.value.scotlandGoals - 1)
}

async function handleSubmit() {
  formError.value = ''

  if (!form.value.name.trim()) {
    formError.value = 'Por favor, informe seu nome completo.'
    return
  }
  if (matchClosed.value) {
    formError.value = 'Os palpites estão encerrados!'
    return
  }

  const nameNorm = form.value.name.toLowerCase().trim()
  const alreadyExists = bets.value.find(b => b.name.toLowerCase().trim() === nameNorm)
  if (alreadyExists) {
    formError.value = 'Esse nome já foi registrado. Cada pessoa pode fazer apenas um palpite.'
    return
  }

  if (currentScoreCount.value >= MAX_SAME) {
    formError.value = `⚠️ O placar ${form.value.brazilGoals}×${form.value.scotlandGoals} já atingiu o limite! Escolha outro.`
    return
  }

  submitting.value = true
  try {
    const newBet = {
      name: form.value.name.trim(),
      brazilGoals: form.value.brazilGoals,
      scotlandGoals: form.value.scotlandGoals,
      status: 'pending',
      createdAt: new Date()
    }
    const docRef = await addDoc(collection(db, 'bets'), newBet)
    pendingBet.value = { id: docRef.id, ...newBet }
    form.value = { name: '', brazilGoals: 0, scotlandGoals: 0 }
  } catch (e) {
    formError.value = 'Erro ao salvar. Tente novamente.'
    console.error(e)
  } finally {
    submitting.value = false
  }
}

// ─── Admin ────────────────────────────────────────────────────
function handleAdminLogin() {
  if (adminInput.value === 'elaine2024') {
    isAdmin.value = true
    adminError.value = ''
  } else {
    adminError.value = 'Senha incorreta.'
  }
}

async function confirmPayment(id) {
  try {
    await updateDoc(doc(db, 'bets', id), { status: 'confirmed' })
  } catch (e) {
    console.error('Erro ao confirmar:', e)
  }
}

// ─── Helpers ──────────────────────────────────────────────────
function formatMoney(value) {
  return value.toFixed(2).replace('.', ',')
}
</script>

<style scoped>
/* ── Variáveis ── */
:root {
  --green: #009C3B;
  --green-dark: #007a2e;
  --green-light: #e8f5e9;
  --yellow: #FFD700;
  --yellow-dark: #c8a600;
  --yellow-light: #fffde7;
  --border: #c8e6c9;
  --border-y: #ffe082;
  --text: #1a2e1a;
  --muted: #5a7a5a;
}

/* ── Layout ── */
.app {
  min-height: 100vh;
  background: linear-gradient(160deg, #e8f5e9, #fffde7, #e8f5e9);
  font-family: 'Segoe UI', system-ui, sans-serif;
  color: var(--text);
}

.main {
  max-width: 480px;
  margin: 0 auto;
  padding: 16px 14px 40px;
}

/* ── Header ── */
.header {
  background: linear-gradient(135deg, #009C3B, #007a2e);
  color: #fff;
  padding: 22px 20px 16px;
  text-align: center;
  border-bottom: 5px solid #FFD700;
}
.header-label {
  font-size: 11px;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: #fff59d;
  font-weight: 700;
  margin-bottom: 6px;
}
.header-teams {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-bottom: 10px;
}
.flag { font-size: 34px; }
.header-vs-box {
  background: rgba(0,0,0,0.2);
  border-radius: 12px;
  padding: 6px 20px;
  border: 2px solid #FFD700;
}
.team-name { font-size: 19px; font-weight: 800; letter-spacing: 1px; }
.vs-text { font-size: 10px; color: #FFD700; margin: 1px 0; font-weight: 700; }
.header-badge {
  background: #FFD700;
  border-radius: 20px;
  display: inline-block;
  padding: 5px 20px;
  font-size: 14px;
  font-weight: 800;
  color: #007a2e;
}
.header-date { font-size: 11px; color: #fff59d; margin-top: 8px; opacity: 0.9; }

/* ── Countdown ── */
.countdown-box {
  background: linear-gradient(135deg, #009C3B, #007a2e);
  color: #fff;
  border-radius: 14px;
  padding: 14px 16px;
  text-align: center;
  border: 3px solid #FFD700;
  margin-bottom: 14px;
}
.countdown-label {
  font-size: 11px;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: #FFD700;
  font-weight: 700;
  margin-bottom: 8px;
}
.countdown-units { display: flex; gap: 8px; justify-content: center; }
.countdown-unit {
  background: rgba(0,0,0,0.2);
  border-radius: 10px;
  padding: 8px 12px;
  min-width: 52px;
  text-align: center;
}
.countdown-value { display: block; font-size: 24px; font-weight: 700; line-height: 1; }
.countdown-sub { display: block; font-size: 10px; opacity: 0.85; margin-top: 2px; }
.countdown-date { font-size: 11px; opacity: 0.8; margin-top: 8px; }
.closed-banner {
  background: #c62828;
  color: #fff;
  border-radius: 12px;
  padding: 12px 20px;
  text-align: center;
  font-weight: 600;
  font-size: 14px;
  margin-bottom: 14px;
}

/* ── Stats ── */
.stats-card {
  background: #fff;
  border-radius: 14px;
  border: 2px solid #FFD700;
  padding: 14px 18px;
  margin-bottom: 14px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.stats-divider { width: 1px; height: 46px; background: #c8e6c9; }
.stats-label { font-size: 11px; color: var(--muted); font-weight: 600; text-transform: uppercase; }
.stats-value { font-size: 26px; font-weight: 800; }
.stats-value.green { color: #009C3B; }
.stats-value.dark { color: #007a2e; }
.stats-sub { font-size: 11px; color: var(--muted); }

/* ── Form ── */
.form-card {
  background: #fff;
  border-radius: 16px;
  border: 2px solid #c8e6c9;
  padding: 18px;
  margin-bottom: 14px;
}
.section-title { font-size: 15px; font-weight: 700; margin: 0 0 14px; color: var(--text); }
.field { margin-bottom: 13px; }
.field-label { display: block; font-size: 12px; font-weight: 600; color: var(--muted); margin-bottom: 5px; }
.field-input {
  width: 100%;
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid #c8e6c9;
  font-size: 14px;
  outline: none;
  background: #e8f5e9;
  color: var(--text);
  font-family: inherit;
}

/* ── Score Picker ── */
.score-picker {
  background: linear-gradient(135deg, #e8f5e9, #fffde7);
  border-radius: 12px;
  padding: 12px 14px;
  border: 1px solid #c8e6c9;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
}
.score-team { text-align: center; flex: 1; }
.score-flag { font-size: 22px; display: block; }
.score-team-name { font-size: 11px; font-weight: 700; display: block; margin-bottom: 6px; }
.score-team-name.green { color: #009C3B; }
.score-team-name.dark { color: #007a2e; }
.score-x { font-size: 20px; font-weight: 800; color: #FFD700; padding-bottom: 14px; }
.goal-input { display: flex; align-items: center; gap: 7px; justify-content: center; }
.goal-btn {
  width: 33px; height: 33px;
  border-radius: 8px;
  border: 1px solid #c8e6c9;
  background: #e8f5e9;
  font-size: 17px;
  cursor: pointer;
  font-weight: 700;
  color: #009C3B;
  display: flex; align-items: center; justify-content: center;
  font-family: inherit;
}
.goal-btn:active { transform: scale(0.93); }
.goal-value {
  width: 40px; height: 33px;
  border-radius: 8px;
  border: 2px solid #FFD700;
  background: #fff;
  display: flex; align-items: center; justify-content: center;
  font-size: 19px; font-weight: 700;
  color: var(--text);
}
.score-warning {
  margin-top: 10px;
  text-align: center;
  font-size: 12px;
  font-weight: 600;
  color: #f57f17;
}
.score-warning.danger { color: #c62828; }

/* ── Erros / Submit ── */
.error-box {
  background: #fff8e1;
  border: 1px solid #ffe082;
  border-radius: 9px;
  padding: 8px 11px;
  font-size: 13px;
  color: #e65100;
  margin-bottom: 12px;
}
.submit-btn {
  width: 100%;
  padding: 13px;
  border-radius: 12px;
  background: linear-gradient(135deg, #009C3B, #007a2e);
  color: #fff;
  border: none;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 3px 0 #007a2e;
  font-family: inherit;
  transition: opacity 0.15s;
}
.submit-btn:active { transform: translateY(2px); box-shadow: none; }
.submit-btn.disabled,
.submit-btn:disabled {
  background: #bdbdbd;
  box-shadow: none;
  cursor: not-allowed;
  opacity: 0.8;
}

/* ── Closed card ── */
.closed-card {
  background: #fff;
  border-radius: 14px;
  border: 2px solid #c8e6c9;
  padding: 16px;
  margin-bottom: 14px;
  text-align: center;
  color: var(--muted);
}
.closed-icon { font-size: 30px; margin-bottom: 6px; }
.closed-title { font-weight: 600; color: var(--text); margin-bottom: 4px; }
.closed-sub { font-size: 13px; }

/* ── Bets ── */
.bets-section { margin-bottom: 14px; }
.bets-header { display: flex; align-items: center; gap: 8px; margin-bottom: 10px; }
.bets-badge {
  background: #FFD700;
  color: #007a2e;
  border-radius: 20px;
  padding: 2px 10px;
  font-size: 12px;
  font-weight: 700;
}
.loading, .empty-state {
  background: #fff;
  border-radius: 12px;
  border: 1px solid #c8e6c9;
  padding: 22px;
  text-align: center;
  color: var(--muted);
  font-size: 14px;
}
.bets-list { display: flex; flex-direction: column; gap: 8px; }
.bet-card {
  background: #fff;
  border: 1px solid #c8e6c9;
  border-left: 4px solid #FFD700;
  border-radius: 12px;
  padding: 11px 13px;
  display: flex;
  align-items: center;
  gap: 10px;
}
.bet-card.confirmed { border-left-color: #009C3B; }
.bet-info { flex: 1; }
.bet-name { font-weight: 600; font-size: 14px; color: var(--text); }
.bet-score { font-size: 13px; color: var(--muted); margin-top: 1px; }
.bet-score strong.green { color: #009C3B; }
.bet-score strong.dark { color: #007a2e; }
.bet-status-col { display: flex; flex-direction: column; align-items: flex-end; gap: 4px; }
.bet-status { font-size: 11px; font-weight: 600; }
.bet-status.confirmed { color: #2e7d32; }
.bet-status.pending { color: #f57f17; }
.confirm-btn {
  font-size: 11px;
  padding: 3px 9px;
  border-radius: 6px;
  border: 1px solid #2e7d32;
  background: transparent;
  color: #2e7d32;
  cursor: pointer;
  font-weight: 600;
  font-family: inherit;
}

/* ── Admin ── */
.admin-card {
  background: #fff;
  border-radius: 14px;
  border: 1px solid #c8e6c9;
  padding: 14px 16px;
  margin-bottom: 8px;
}
.admin-label { font-size: 12px; font-weight: 600; color: var(--muted); margin-bottom: 7px; }
.admin-row { display: flex; gap: 8px; }
.admin-btn {
  padding: 9px 14px;
  border-radius: 8px;
  background: #009C3B;
  color: #fff;
  border: none;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  font-family: inherit;
  white-space: nowrap;
}
.admin-error { color: #c62828; font-size: 12px; margin-top: 5px; }
.admin-logged { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; }
.admin-logged-title { font-size: 13px; font-weight: 700; color: #2e7d32; flex: 1; }
.admin-logout {
  font-size: 12px; color: var(--muted); background: none; border: none; cursor: pointer; font-family: inherit;
}
.admin-hint { font-size: 12px; color: var(--muted); width: 100%; }

/* ── Footer ── */
.footer { text-align: center; margin-top: 26px; font-size: 12px; color: var(--muted); }

/* ── Modal ── */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.6);
  z-index: 999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
}
.modal {
  background: #fff;
  border-radius: 20px;
  padding: 24px 20px;
  max-width: 360px;
  width: 100%;
  text-align: center;
  border: 3px solid #FFD700;
}
.modal-icon { font-size: 38px; margin-bottom: 6px; }
.modal-title { font-size: 18px; font-weight: 700; margin: 0 0 6px; color: var(--text); }
.modal-sub { color: var(--muted); font-size: 13px; margin: 0 0 16px; }
.pix-box {
  background: #e8f5e9;
  border-radius: 12px;
  padding: 14px;
  margin-bottom: 14px;
  border: 1px solid #c8e6c9;
}
.pix-value { font-size: 30px; font-weight: 800; color: #009C3B; margin-bottom: 4px; }
.pix-label { font-size: 12px; color: var(--muted); margin-bottom: 8px; }
.pix-key {
  background: #fff;
  border: 2px dashed #009C3B;
  border-radius: 10px;
  padding: 8px 12px;
  font-size: 17px;
  font-weight: 700;
  letter-spacing: 1px;
}
.pix-name { font-size: 12px; color: var(--muted); margin-top: 6px; }
.modal-tip {
  background: #fffde7;
  border: 1px solid #ffe082;
  border-radius: 10px;
  padding: 10px 12px;
  font-size: 12px;
  color: #6d4c00;
  margin-bottom: 14px;
  text-align: left;
}
.modal-bet-info {
  background: #e8f5e9;
  border-radius: 10px;
  padding: 10px 12px;
  font-size: 13px;
  margin-bottom: 16px;
  text-align: left;
  border: 1px solid #c8e6c9;
}
.modal-bet-title { font-weight: 600; margin-bottom: 4px; color: var(--text); }
.modal-bet-name { color: var(--muted); margin-top: 2px; }

/* ── Helpers ── */
.green { color: #009C3B; }
.dark { color: #007a2e; }
</style>
