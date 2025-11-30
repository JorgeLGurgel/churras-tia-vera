<script setup>
import { ref, computed, reactive } from 'vue';

// --- CONFIGURAÇÕES ---
const PHONE_NUMBER = '5592992885358'; // SEU NÚMERO AQUI

// --- DADOS DO CARDÁPIO ---
const menu = ref([
  {
    category: 'Espetinho Simples',
    description: 'Acompanha Farofa',
    items: [
      { id: 1, name: 'Carne (Alcatra)', price: 16.90 },
      { id: 2, name: 'Misto', price: 14.90 },
      { id: 3, name: 'Frango', price: 14.90 },
      { id: 4, name: 'Filé com Bacon', price: 21.90 },
    ]
  },
  {
    category: 'Espetinho Completo',
    description: 'Acompanha Arroz com Brócolis, Farofa e Vinagrete',
    items: [
      { id: 5, name: 'Carne (Alcatra)', price: 26.90 },
      { id: 6, name: 'Misto', price: 24.90 },
      { id: 7, name: 'Frango', price: 24.90 },
      { id: 8, name: 'Filé com Bacon', price: 31.90 },
    ]
  },
  {
    category: 'Guarnições',
    description: 'Porções extras',
    items: [
      { id: 9, name: 'Arroz Branco', price: 7.00 },
      { id: 10, name: 'Arroz com Brócolis', price: 8.00 },
      { id: 11, name: 'Farofa', price: 2.50 },
      { id: 12, name: 'Vatapá', price: 9.90 },
      { id: 13, name: 'Vinagrete', price: 4.90 },
    ]
  },
  {
    category: 'Bebidas & Sobremesas',
    description: '',
    items: [
      { id: 14, name: 'Refri Lata', price: 7.00 },
      { id: 15, name: 'Pudim', price: 8.00 },
    ]
  }
]);

// --- ESTADO ---
const cart = ref([]);
const isCheckoutOpen = ref(false);
const isLoadingCep = ref(false);

// --- DADOS DO CLIENTE ATUALIZADOS ---
const customer = reactive({
  name: '',
  condo: '',     // Novo: Armazena qual condomínio
  street: '',    // Novo: Rua do condomínio
  number: '',    // Número da casa
  paymentMethod: '',
  changeFor: ''  // Novo: Valor para troco
});

// Opções de Pagamento com Ícones
const paymentOptions = [
  { id: 'PIX', label: 'PIX', icon: '💠' },
  { id: 'Dinheiro', label: 'Dinheiro', icon: '💵' },
  // { id: 'Cartão Crédito', label: 'Crédito', icon: '💳' },
  // { id: 'Cartão Débito', label: 'Débito', icon: '🏧' }
];

// --- LÓGICA DO CARRINHO ---
const addToCart = (item) => {
  const existingItem = cart.value.find(i => i.id === item.id);
  if (existingItem) {
    existingItem.quantity++;
  } else {
    const parentCategory = menu.value.find(cat => cat.items.find(i => i.id === item.id)).category;
    cart.value.push({ ...item, quantity: 1, categoryName: parentCategory });
  }
};

const increaseQtd = (item) => {
  item.quantity++;
};

const decreaseQtd = (item) => {
  if (item.quantity > 1) {
    item.quantity--;
  }
};

const deleteItem = (item) => {
  const index = cart.value.indexOf(item);
  if (index > -1) {
    cart.value.splice(index, 1);
  }
};

const cartTotal = computed(() => {
  return cart.value.reduce((acc, item) => acc + (item.price * item.quantity), 0);
});

const cartCount = computed(() => {
  return cart.value.reduce((acc, item) => acc + item.quantity, 0);
});

// --- API DE CEP ---
// --- API DE CEP ---
const searchCep = async () => {
  const cleanCep = customer.cep.replace(/\D/g, '');
  
  // Se o usuário apagar o CEP, limpamos o endereço e os campos somem
  if (cleanCep === '') {
    customer.address = '';
    customer.number = '';
    customer.complement = '';
    return;
  }

  if (cleanCep.length === 8) {
    isLoadingCep.value = true;
    customer.address = ''; // Limpa o endereço anterior para "resetar" a visão
    
    try {
      const response = await fetch(`https://viacep.com.br/ws/${cleanCep}/json/`);
      const data = await response.json();
      
      if (!data.erro) {
        customer.address = `${data.logradouro}, ${data.bairro}`;
        
        // Pequeno delay para o DOM atualizar e o campo aparecer antes de focar
        setTimeout(() => {
            document.getElementById('numberInput')?.focus();
        }, 100);
      } else {
        alert('CEP não encontrado. Verifique o número.');
      }
    } catch (error) {
      console.error(error);
    } finally {
      isLoadingCep.value = false;
    }
  }
};

// --- VALIDAÇÃO ATUALIZADA ---
const isOrderValid = computed(() => {
  if (cart.value.length === 0) return false;
  if (!customer.name || customer.name.trim() === '') return false;
  
  // Validações do novo endereço
  if (!customer.condo) return false;
  if (!customer.street || customer.street.trim() === '') return false;
  if (!customer.number || customer.number.trim() === '') return false;

  if (!customer.paymentMethod) return false;

  return true;
});

// --- ENVIO WHATSAPP ATUALIZADO ---
const sendOrder = () => {
  if (!isOrderValid.value) return;

  let text = `🔥 *PEDIDO - TIA VERA* 🔥\n\n`;
  text += `👤 *Cliente:* ${customer.name}\n`;
  text += `-----------------------------------\n`;
  
  // Itens
  text += `📋 *ITENS:* \n`;
  cart.value.forEach(item => {
    const totalItem = (item.price * item.quantity).toFixed(2).replace('.', ',');
    let nomeFormatado = item.name;

    if (item.categoryName.includes('Completo')) nomeFormatado += ' ⭐ (COMPLETO)'; 
    else if (item.categoryName.includes('Simples')) nomeFormatado += ' (Simples)';

    text += `▪️ ${item.quantity}x ${nomeFormatado} ... R$ ${totalItem}\n`;
  });

  text += `\n💰 *TOTAL FINAL: R$ ${cartTotal.value.toFixed(2).replace('.', ',')}*\n`;
  text += `-----------------------------------\n`;
  
  // Endereço Novo
  text += `📍 *ENTREGA:* \n`;
  text += `🏢 ${customer.condo}\n`;
  text += `Rua: ${customer.street}\n`;
  text += `Casa Nº: ${customer.number}\n\n`;
  
  // Pagamento e Troco
  text += `💳 *PAGAMENTO:* ${customer.paymentMethod}`;
  
  if (customer.paymentMethod === 'Dinheiro') {
    if (customer.changeFor) {
      text += `\n💵 (Troco para: ${customer.changeFor})`;
    } else {
      text += `\n💵 (Não precisa de troco)`;
    }
  }

  const url = `https://wa.me/${PHONE_NUMBER}?text=${encodeURIComponent(text)}`;
  window.open(url, '_blank');
};


const formatCurrency = (value) => {
  return value.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
};
</script>

<template>
  <div class="app-container">
    
    <!-- HEADER -->
    <header class="hero">
      <div class="logo-area">
        <div class="logo-circle">
  <!-- Caminho da imagem considerando que ela está na pasta public -->
  <img src="/logo.png" alt="Logo Tia Vera" class="logo-img" />
</div>
        <h1>Churrasquinho<br>da Tia Vera</h1>
      </div>
    </header>

    <!-- MENU -->
    <main class="menu-list">
      <div v-for="(category, index) in menu" :key="index" class="category-section">
        <h2 class="category-title">{{ category.category }}</h2>
        <p v-if="category.description" class="category-desc">
          {{ category.description }}
        </p>

        <div class="items-grid">
          <div v-for="item in category.items" :key="item.id" class="menu-item">
            <div class="item-info">
              <h3>{{ item.name }}</h3>
              <span class="price">{{ formatCurrency(item.price) }}</span>
            </div>
            <button @click="addToCart(item)" class="add-btn">ADICIONAR</button>
          </div>
        </div>
      </div>
    </main>

    <!-- FOOTER / CART BUTTON -->
    <transition name="slide-up">
      <div class="floating-cart" v-if="cart.length > 0">
        <div class="cart-info">
          <span class="cart-count-badge">{{ cartCount }}</span>
          <span class="cart-total-text">Total: {{ formatCurrency(cartTotal) }}</span>
        </div>
        <button @click="isCheckoutOpen = true" class="view-cart-btn">
          Ver Carrinho
        </button>
      </div>
    </transition>

    <!-- MODAL CHECKOUT -->
    <div v-if="isCheckoutOpen" class="modal-overlay">
      <div class="modal-content">
        <div class="modal-header">
          <h2>Finalizar Pedido</h2>
          <button @click="isCheckoutOpen = false" class="close-btn">✕</button>
        </div>

        <div class="modal-body">
          <!-- === SEÇÃO 1: RESUMO DO PEDIDO (Recuperado) === -->
          <h3 class="section-title">Seu Pedido</h3>
          
          <ul class="cart-items-list">
            <li v-for="item in cart" :key="item.id" class="cart-item-row">
              <div class="item-left">
                <div class="item-name">
                  {{ item.name }}
                  <!-- Mostra se é Completo ou Simples visualmente -->
                  <span v-if="item.categoryName.includes('Completo')" class="badge-complete">⭐ Completo</span>
                  <span v-else class="badge-simple">Simples</span>
                </div>
                <div class="item-price-unit">{{ formatCurrency(item.price) }} / un</div>
              </div>
              
              <div class="item-right">
                <div class="qty-controls">
                  <button @click="decreaseQtd(item)" class="qty-btn" :disabled="item.quantity <= 1">-</button>
                  <span class="qty-display">{{ item.quantity }}</span>
                  <button @click="increaseQtd(item)" class="qty-btn">+</button>
                </div>
                <button @click="deleteItem(item)" class="trash-btn">
                  🗑️
                </button>
              </div>
            </li>
          </ul>

          <!-- SUBTOTAL -->
          <div class="total-row">
            <span>Total a Pagar:</span>
            <span class="total-price">{{ formatCurrency(cartTotal) }}</span>
          </div>

          <hr class="divider">
         <!-- FORMULÁRIO -->
          <form @submit.prevent="sendOrder" class="checkout-form">
            
            <h3 class="section-title">Dados de Entrega</h3>
            
            <div class="form-group">
              <label>Nome Completo</label>
              <input v-model="customer.name" placeholder="Ex: Jorge Silva" required />
            </div>

            <!-- SELEÇÃO DE CONDOMÍNIO -->
            <div class="form-group">
              <label>Onde será a entrega?</label>
              <div class="condo-grid">
                <div 
                  class="condo-card"
                  :class="{ active: customer.condo === 'Condomínio Vila Gaia' }"
                  @click="customer.condo = 'Condomínio Vila Gaia'"
                >
                  🏢 Vila Gaia
                </div>
                <div 
                  class="condo-card"
                  :class="{ active: customer.condo === 'Condomínio Estoril' }"
                  @click="customer.condo = 'Condomínio Estoril'"
                >
                  🏢 Estoril
                </div>
              </div>
            </div>

            <!-- RUA E NÚMERO (Só aparecem se escolher o condomínio) -->
            <div v-if="customer.condo" class="slide-in">
              <p class="selected-condo-text">Entregar em: <strong>{{ customer.condo }}</strong></p>
              
              <div class="form-group">
                <label>Nome da Rua</label>
                <input v-model="customer.street" placeholder="Ex: Rua A" required />
              </div>

              <div class="form-group">
                <label>Número da Casa</label>
                <input v-model="customer.number" placeholder="Nº da casa" type="tel" required />
              </div>
            </div>

            <h3 class="section-title">Forma de Pagamento</h3>
            
            <div class="payment-grid">
              <div 
                v-for="opt in paymentOptions" 
                :key="opt.id"
                class="payment-card"
                :class="{ active: customer.paymentMethod === opt.id }"
                @click="customer.paymentMethod = opt.id"
              >
                <span class="pay-icon">{{ opt.icon }}</span>
                <span class="pay-label">{{ opt.label }}</span>
              </div>
            </div>

            <!-- CAMPO DE TROCO (Só aparece se for Dinheiro) -->
            <div v-if="customer.paymentMethod === 'Dinheiro'" class="change-box slide-in">
              <label>Precisa de troco para quanto?</label>
              <input 
                v-model="customer.changeFor" 
                v-money-format
                placeholder="Ex: 50,00 (Deixe vazio se for certinho)" 
                type="tel"
              />
            </div>

            <button 
              type="submit" 
              class="whatsapp-btn" 
              :disabled="!isOrderValid"
            >
              <span v-if="!isOrderValid" class="icon">🔒</span>
              <span v-else class="icon">📲</span>
              
              <span v-if="!isOrderValid">Preencha os dados obrigatórios</span>
              <span v-else>Enviar Pedido no WhatsApp</span>
            </button>
          </form>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
/* CORES DA MARCA */
:root {
  --brand-red: #A91B0E;
  --brand-yellow: #FFC107;
  --bg-dark: #1a0505;
  --text-light: #ffffff;
  --input-bg: #333; /* Fundo dos inputs escuro */
  --input-border: #555;
}

* { box-sizing: border-box; font-family: 'Segoe UI', sans-serif; }

.app-container {
  background-color: #121212;
  color: #fff;
  min-height: 100vh;
  padding-bottom: 100px;
  max-width: 600px;
  margin: 0 auto;
}

/* HEADER */
.hero {
  background: linear-gradient(180deg, #A91B0E 0%, #610e05 100%);
  padding: 25px;
  text-align: center;
  border-bottom-left-radius: 25px;
  border-bottom-right-radius: 25px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.5);
}

.logo-circle {
  width: 80px;  /* Aumentei um pouquinho para destacar mais */
  height: 80px;
  background: #FFC107;
  border-radius: 50%;
  margin: 0 auto 10px;
  border: 3px solid #fff;
  /* Isso garante que a imagem não vaze do círculo */
  overflow: hidden; 
  /* Removemos o display flex antigo pois agora é uma imagem cheia */
  display: block; 
}

/* Adicione esta nova classe */
.logo-img {
  width: 100%;
  height: 100%;
  object-fit: cover; /* Faz a imagem preencher tudo sem esticar */
  display: block;
}

h1 { margin: 0; color: #FFC107; text-transform: uppercase; font-size: 1.4rem; text-shadow: 1px 1px 2px #000; }

/* MENU */
.menu-list { padding: 20px; }
.category-section { margin-bottom: 25px; }
.category-title { color: #FFC107; font-size: 1.2rem; border-left: 4px solid #A91B0E; padding-left: 10px; margin-bottom: 5px; }
.category-desc { color: #aaa; font-size: 0.9rem; margin-bottom: 15px; margin-left: 14px; }

.menu-item {
  background: #1e1e1e;
  padding: 15px;
  border-radius: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  border: 1px solid #333;
}
.item-info h3 { margin: 0 0 5px; font-size: 1rem; color: #fff; }
.price { color: #FFC107; font-weight: bold; }

.add-btn {
  background: #A91B0E; color: #fff; border: none;
  padding: 8px 15px; border-radius: 20px;
  font-weight: bold; font-size: 0.8rem;
  cursor: pointer;
}

/* CARRINHO FLUTUANTE */
.floating-cart {
  position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%);
  width: 90%; max-width: 550px;
  background: #FFC107; color: #000;
  padding: 15px 25px; border-radius: 50px;
  display: flex; justify-content: space-between; align-items: center;
  box-shadow: 0 10px 25px rgba(0,0,0,0.6);
  z-index: 99;
}
.cart-total-text { font-weight: 800; font-size: 1.1rem; }
.cart-count-badge { background: #A91B0E; color: #fff; padding: 2px 8px; border-radius: 10px; font-size: 0.8rem; margin-right: 5px; font-weight: bold; }
.view-cart-btn { background: #000; color: #FFC107; border: none; padding: 8px 16px; border-radius: 20px; font-weight: bold; cursor: pointer; }

/* MODAL */
.modal-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.85);
  display: flex; align-items: flex-end; justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #1e1e1e; /* Fundo escuro do modal */
  color: #fff;
  width: 100%; max-width: 600px; height: 90vh;
  border-top-left-radius: 20px; border-top-right-radius: 20px;
  display: flex; flex-direction: column;
}

.modal-header {
  padding: 20px; border-bottom: 1px solid #333;
  display: flex; justify-content: space-between; align-items: center;
}
.close-btn { background: none; border: none; color: #fff; font-size: 1.5rem; cursor: pointer; }

.modal-body {
  padding: 20px; overflow-y: auto; flex: 1;
}

/* Divisor entre lista e formulário */
.divider {
  border: 0;
  border-top: 1px dashed #444;
  margin: 20px 0;
}

/* Badges para diferenciar Simples de Completo na lista */
.badge-complete {
  background-color: #FFC107;
  color: #000;
  font-size: 0.7rem;
  padding: 2px 6px;
  border-radius: 4px;
  margin-left: 5px;
  font-weight: bold;
  vertical-align: middle;
}

.badge-simple {
  background-color: #444;
  color: #ccc;
  font-size: 0.7rem;
  padding: 2px 6px;
  border-radius: 4px;
  margin-left: 5px;
  vertical-align: middle;
}

/* ESTILOS NOVOS PARA CONDOMÍNIO */
.condo-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-bottom: 15px;
}

.condo-card {
  background: #333;
  border: 2px solid #444;
  padding: 15px 5px;
  border-radius: 10px;
  text-align: center;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
  color: #ddd;
}

.condo-card.active {
  border-color: #FFC107;
  background: rgba(255, 193, 7, 0.15);
  color: #FFC107;
}

.selected-condo-text {
  color: #FFC107;
  margin-bottom: 10px;
  font-size: 0.9rem;
  border-bottom: 1px dashed #555;
  padding-bottom: 5px;
}

/* ESTILO DO CAMPO DE TROCO */
.change-box {
  background: #2a2a2a;
  padding: 15px;
  border-radius: 8px;
  margin-top: -20px; /* Cola no grid de pagamento */
  margin-bottom: 20px;
  border: 1px solid #444;
  border-left: 4px solid #FFC107;
}

.change-box label {
  color: #FFC107;
  font-weight: bold;
}

.section-title {
  color: #FFC107; margin: 25px 0 15px; font-size: 1.1rem; text-transform: uppercase; letter-spacing: 1px;
}
.section-title:first-child { margin-top: 0; }

/* LISTA DE ITENS NO MODAL */
.cart-items-list { list-style: none; padding: 0; margin: 0; }
.cart-item-row {
  display: flex; justify-content: space-between; align-items: center;
  padding: 15px 0; border-bottom: 1px solid #333;
}
.item-left { flex: 1; }
.item-name { font-weight: bold; margin-bottom: 4px; }
.item-price-unit { font-size: 0.85rem; color: #aaa; }

.item-right { display: flex; align-items: center; gap: 10px; }
.qty-controls {
  display: flex; align-items: center; background: #333;
  border-radius: 8px; overflow: hidden;
}
.qty-btn {
  background: transparent; color: #fff; border: none;
  width: 30px; height: 30px; font-size: 1.2rem; cursor: pointer;
  display: flex; align-items: center; justify-content: center;
}
.qty-btn:disabled { opacity: 0.3; }
.qty-display { padding: 0 5px; font-weight: bold; }

.trash-btn {
  background: rgba(255, 0, 0, 0.2); color: #ff4d4d;
  border: none; width: 35px; height: 35px; border-radius: 8px;
  cursor: pointer; display: flex; align-items: center; justify-content: center;
  font-size: 1.1rem; transition: background 0.2s;
}
.trash-btn:hover { background: rgba(255, 0, 0, 0.4); }

.total-row {
  display: flex; justify-content: space-between; align-items: center;
  margin-top: 20px; padding-top: 20px; border-top: 1px dashed #555;
  font-size: 1.3rem; font-weight: bold; color: #FFC107;
}

/* FORMULÁRIO */
.form-group { margin-bottom: 15px; }
.form-group label { display: block; margin-bottom: 5px; color: #aaa; font-size: 0.9rem; }
.form-row { display: flex; gap: 15px; }
.short { width: 80px; }

.checkout-form input {
  width: 100%; background: #333; border: 1px solid #444; color: #fff;
  padding: 12px; border-radius: 8px; font-size: 1rem; outline: none;
}
.checkout-form input:focus { border-color: #FFC107; }

.input-wrapper { position: relative; }
.loading-icon { position: absolute; right: 10px; top: 12px; }

.address-box {
  background: rgba(255, 193, 7, 0.1); border: 1px solid #FFC107;
  padding: 10px; border-radius: 8px; margin-bottom: 15px; color: #FFC107;
}

/* PAGAMENTO GRID */
.payment-grid {
  display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 30px;
}
.payment-card {
  background: #333; border: 2px solid transparent;
  padding: 15px; border-radius: 10px; cursor: pointer;
  display: flex; flex-direction: column; align-items: center; gap: 5px;
  transition: all 0.2s;
}
.payment-card.active {
  border-color: #FFC107; background: rgba(255, 193, 7, 0.1);
}
.pay-icon { font-size: 1.5rem; }
.pay-label { font-size: 0.9rem; }

.whatsapp-btn {
  width: 100%; background: #25D366; color: #fff;
  padding: 16px; border: none; border-radius: 10px;
  font-size: 1.1rem; font-weight: bold; cursor: pointer;
  display: flex; align-items: center; justify-content: center; gap: 10px;
}
.whatsapp-btn:hover { background: #1ebc57; }

/* Estado Desativado do Botão */
.whatsapp-btn:disabled {
  background-color: #444; /* Cinza escuro */
  color: #888; /* Texto cinza claro */
  cursor: not-allowed; /* Mostra o símbolo de proibido no mouse */
  border: 1px solid #555;
  opacity: 0.8;
}

/* Remove o efeito de hover quando desativado */
.whatsapp-btn:disabled:hover {
  background-color: #444;
}

@media (min-width: 600px) {
  .modal-overlay { align-items: center; }
  .modal-content { height: auto; max-height: 90vh; border-radius: 20px; }
}

/* Container que segura o input e o botão juntos */
.cep-input-group {
  display: flex;
  width: 100%;
}

/* Ajuste no input para ele não ter bordas arredondadas na direita */
.cep-input-group input {
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
  border-right: none; /* Remove a borda da direita para colar no botão */
  flex: 1; /* Ocupa todo o espaço possível */
}

/* Estilo do Botão de Busca */
.search-btn {
  background-color: #FFC107; /* Amarelo da marca */
  border: 1px solid #FFC107;
  border-top-right-radius: 8px;
  border-bottom-right-radius: 8px;
  width: 50px;
  cursor: pointer;
  font-size: 1.2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.search-btn:active {
  background-color: #e0a800;
}

.search-btn:disabled {
  background-color: #ccc;
  border-color: #ccc;
  cursor: not-allowed;
}

.slide-in {
  animation: slideDown 0.4s ease-out forwards;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>