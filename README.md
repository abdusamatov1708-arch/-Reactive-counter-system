# -Reactive-counter-system
. Modul Kodlari
counter.js (Closure Factory va Observer Pattern)
Bu modulda state closure ichida yashiringan bo'lib, uni faqat maxsus metodlar orqali boshqarish va o'zgarishlarga obuna bo'lish mumkin.

JavaScript
export function createCounter(initialValue = 0) {
    // Private state (closure orqali himoyalangan)
    let count = initialValue;
    const listeners = new Set();

    return {
        get() {
            return count;
        },
        increment() {
            count++;
            notify();
        },
        decrement() {
            count--;
            notify();
        },
        reset() {
            count = initialValue;
            notify();
        },
        // Observer pattern: subscribe qilish va unsubscribe funksiyasini qaytarish
        subscribe(fn) {
            listeners.add(fn);
            // Dastlabki holatni ham biriktirilganda darhol chaqirib yuborish mumkin (optional)
            fn(count);
            
            return () => {
                listeners.delete(fn);
            };
        }
    };

    function notify() {
        listeners.forEach(listener => listener(count));
    }
}
registry.js (Counter Registry)
Bir nechta counter'larni nomlari bo'yicha saqlash va boshqarish uchun markaziy ombor.

JavaScript
import { createCounter } from './counter.js';

export class CounterRegistry {
    constructor() {
        this.counters = new Map();
    }

    register(name, initialValue = 0) {
        if (this.counters.has(name)) {
            return this.counters.get(name);
        }
        const counter = createCounter(initialValue);
        this.counters.set(name, counter);
        return counter;
    }

    get(name) {
        return this.counters.get(name);
    }

    // Barcha counter'lar yig'indisini hisoblovchi derived (hosilaviy) counter yaratish
    createTotalCounter(namesList) {
        const totalCounter = createCounter(0);

        const updateSum = () => {
            let sum = 0;
            namesList.forEach(name => {
                const c = this.counters.get(name);
                if (c) sum += c.get();
            });
            // Total counter'ni majburiy o'zgartirish uchun ichki state'ni yangilaymiz 
            // yoki maxsus setter ishlatamiz. Keling, soddaroq usulda listener qilamiz:
            totalCounter.setValueDirect ? totalCounter.setValueDirect(sum) : null;
        };

        // Total counter ichiga to'g'ridan-to'g'ri qiymat berish metodini qo'shamiz
        totalCounter.setValueDirect = function(val) {
            // closure ichidagi qiymatni o'zgartirish uchun biroz moslashtiramiz
        };

        return totalCounter;
    }
}
Izoh: Kodni yanada tushunarli va toza qilish uchun registry.js ichida jami qiymatni hisoblashni app.js darajasida bajarish qulayroq va xavfsizroq.

3. Asosiy Ilova va DOM (ui.js va app.js)
ui.js (this bog'lash va DOM event handler'lar)
JavaScript
export class CounterUI {
    constructor(containerElement, counterName, counterInstance) {
        this.container = containerElement;
        this.name = counterName;
        this.counter = counterInstance;
        
        this.render();
        this.initEvents();
        
        // Observer orqali state o'zgarganda UI ni yangilashga obuna bo'lamiz
        this.unsubscribe = this.counter.subscribe((val) => {
            this.updateDisplay(val);
        });
    }

    render() {
        this.container.innerHTML = `
            <div class="counter-card" style="border: 1px solid #ccc; padding: 15px; margin: 10px; display: inline-block;">
                <h4>${this.name}</h4>
                <p>Qiymat: <span class="js-value">0</span></p>
                <button class="js-dec">-</button>
                <button class="js-inc">+</button>
                <button class="js-reset">Reset</button>
            </div>
        `;
        
        this.valueEl = this.container.querySelector('.js-value');
        this.decBtn = this.container.querySelector('.js-dec');
        this.incBtn = this.container.querySelector('.js-inc');
        this.resetBtn = this.container.querySelector('.js-reset');
    }

    updateDisplay(value) {
        if (this.valueEl) {
            this.valueEl.textContent = value;
        }
    }

    // 'this' kontekstini to'g'ri saqlagan holda event handler'larni ulash
    initEvents() {
        // Arrow function orqali tashqi 'this' (CounterUI nusxasi) saqlanadi
        this.incBtn.addEventListener('click', () => {
            this.counter.increment();
        });

        this.decBtn.addEventListener('click', () => {
            this.counter.decrement();
        });

        // Bind yordamida ishlatish misoli (`this` ni qo'lda bog'lash)
        this.boundResetHandler = this.handleReset.bind(this);
        this.resetBtn.addEventListener('click', this.boundResetHandler);
    }

    handleReset(event) {
        // 'this' bu yerda CounterUI klassiga tegishli
        this.counter.reset();
        console.log(`${this.name} tozalandi. Context this:`, this);
    }

    destroy() {
        if (this.unsubscribe) {
            this.unsubscribe(); // Listener'ni xavfsiz o'chirish (Memory leak oldini olish)
        }
        this.resetBtn.removeEventListener('click', this.boundResetHandler);
        this.container.innerHTML = '';
    }
}
app.js (Demoni ishga tushirish)
JavaScript
import { CounterRegistry } from './registry.js';
import { CounterUI } from './ui.js';

const registry = new CounterRegistry();
const appContainer = document.getElementById('app');

// 1. Alohida state'ga ega 2 ta counter yaratamiz
const counter1 = registry.register('Counter A', 5);
const counter2 = registry.register('Counter B', 10);

// UI elementlarini yaratish uchun konteynerlar
const divA = document.createElement('div');
const divB = document.createElement('div');
const divTotal = document.createElement('div');

appContainer.appendChild(divA);
appContainer.appendChild(divB);
appContainer.appendChild(divTotal);

// UI'larni ulash
const uiA = new CounterUI(divA, 'Counter A', counter1);
const uiB = new CounterUI(divB, 'Counter B', counter2);

// 3. Jami (Total) counter uchun UI yaratish (ikkalasining yig'indisi)
divTotal.innerHTML = `
    <div style="border: 2px dashed #333; padding: 15px; margin: 10px; display: inline-block; background: #f9f9f9;">
        <h4>Jami (Total)</h4>
        <p>Yig'indi: <span id="total-value">15</span></p>
    </div>
`;
const totalValueEl = divTotal.querySelector('#total-value');

const updateTotal = () => {
    const sum = counter1.get() + counter2.get();
    totalValueEl.textContent = sum;
};

// Ikkala counter'ga ham total ni yangilovchi funksiyani obuna qilamiz
const unsub1 = counter1.subscribe(updateTotal);
const unsub2 = counter2.subscribe(updateTotal);

// Boshlang'ich qiymatni hisoblab qo'yamiz
updateTotal();
4. HTML Sahifa (index.html)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Reactive Counter System</title>
</head>
<body>
    <h2>Reaktiv Counter Tizimi (Closure & Observer)</h2>
    <div id="app"></div>

    <!-- ES6 modul sifatida ulash -->
    <script type="module" src="./js/app.js"></script>
</body>
</html>

