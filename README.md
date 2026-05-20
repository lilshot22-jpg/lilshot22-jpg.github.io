---
layout: null
---
    <script>
        // Инициализация Telegram Web App
        if (window.Telegram && window.Telegram.WebApp) {
            const tg = window.Telegram.WebApp;
            tg.ready();
            tg.expand();
            tg.backgroundColor = "#0a0a0c";
            tg.headerColor = "#0a0a0c";
        }

        // ЗАЩИТА ОТ СТАРОГО КЭША: Жесткая проверка структуры данных
        let data;
        try {
            let cached = localStorage.getItem('flow13_data');
            data = JSON.parse(cached);
            // Если кэш пустой или остался от старой версии приложения — сбрасываем его
            if (!data || !data.balances || !data.settings || !data.balances.hold) {
                throw new Error("Ресет устаревшего кэша");
            }
        } catch (e) {
            // Инициализация чистых дефолтных данных
            data = {
                balances: { hold: 0, flow: 0, pump: 0 },
                settings: { hold: 30, flow: 50, pump: 20 },
                history: []
            };
        }

        // Форматирование чисел в красивую валюту
        function formatCurrency(num) {
            return new Intl.NumberFormat('ru-RU').format(Math.round(num)) + ' ₽';
        }

        // Обновление всего интерфейса
        function updateUI() {
            // Балансы сейфов
            document.getElementById('bal-hold').innerText = formatCurrency(data.balances.hold);
            document.getElementById('bal-flow').innerText = formatCurrency(data.balances.flow);
            document.getElementById('bal-pump').innerText = formatCurrency(data.balances.pump);

            // Общий баланс
            const total = data.balances.hold + data.balances.flow + data.balances.pump;
            document.getElementById('total-balance').innerText = formatCurrency(total);

            // Проценты в бейджах
            document.getElementById('badge-hold').innerText = data.settings.hold + '%';
            document.getElementById('badge-flow').innerText = data.settings.flow + '%';
            document.getElementById('badge-pump').innerText = data.settings.pump + '%';

            // Синхронизация инпутов настроек
            document.getElementById('set-hold').value = data.settings.hold;
            document.getElementById('set-flow').value = data.settings.flow;
            document.getElementById('set-pump').value = data.settings.pump;

            // Рендер истории
            const histList = document.getElementById('history-list');
            if (!data.history || data.history.length === 0) {
                histList.innerHTML = `<div class="history-item" style="color:#444; justify-content:center;">История пуста</div>`;
            } else {
                histList.innerHTML = data.history.map(item => `
                    <div class="history-item">
                        <span style="color:#aaa;">${item.desc}</span>
                        <span class="${item.type === 'inc' ? 'hist-inc' : 'hist-exp'}">
                            ${item.type === 'inc' ? '+' : '−'} ${formatCurrency(item.amount)}
                        </span>
                    </div>
                `).join('');
            }

            // Сохранение в память телефона
            localStorage.setItem('flow13_data', JSON.stringify(data));
        }

        // Переключение видимости окон ввода
        function toggleModal(modalId) {
            const modals = ['modal-income', 'modal-expense', 'modal-settings'];
            modals.forEach(id => {
                const el = document.getElementById(id);
                if(id === modalId) {
                    el.classList.toggle('hidden');
                } else {
                    el.classList.add('hidden');
                }
            });
        }

        // Логика поступления (Авто-распределение)
        function executeIncome() {
            const input = document.getElementById('income-amount');
            const amount = parseFloat(input.value);
            
            // ТЕПЕРЬ С ALERT: Кнопка больше не будет молчать при ошибке
            if (!amount || amount <= 0) {
                if(window.Telegram && window.Telegram.WebApp) {
                    window.Telegram.WebApp.showAlert("Введи сумму больше нуля, бро!");
                } else {
                    alert("Введи сумму больше нуля, бро!");
                }
                return;
            }

            // Считаем доли на основе настроек матрицы
            const hShare = amount * (data.settings.hold / 100);
            const fShare = amount * (data.settings.flow / 100);
            const pShare = amount * (data.settings.pump / 100);

            // Начисляем
            data.balances.hold += hShare;
            data.balances.flow += fShare;
            data.balances.pump += pShare;

            // Запись в историю логов
            addLog('inc', amount, 'Вливание в матрицу');

            // Сброс поля и скрытие окна
            input.value = '';
            toggleModal('');
            updateUI();
        }

        // Логика списания расходов
        function executeExpense() {
            const input = document.getElementById('expense-amount');
            const target = document.getElementById('expense-target').value;
            const amount = parseFloat(input.value);

            if (!amount || amount <= 0) {
                if(window.Telegram && window.Telegram.WebApp) {
                    window.Telegram.WebApp.showAlert("Укажи сумму расхода.");
                } else {
                    alert("Укажи сумму расхода.");
                }
                return;
            }
            
            if (data.balances[target] < amount) {
                if(window.Telegram && window.Telegram.WebApp) {
                    window.Telegram.WebApp.showAlert("Недостаточно средств в этом сейфе, бро. Сделай перерасчет.");
                } else {
                    alert("Недостаточно средств в этом сейфе, бро. Сделай перерасчет.");
                }
                return;
            }

            // Списываем
            data.balances[target] -= amount;

            const targetNames = { hold: 'Списание из HOLD', flow: 'Расход FLOW', pump: 'Инвестиция PUMP' };
            addLog('exp', amount, targetNames[target]);

            input.value = '';
            toggleModal('');
            updateUI();
        }

        // Сохранение новых пропорций матрицы распределения
        function saveSettings() {
            const h = parseInt(document.getElementById('set-hold').value) || 0;
            const f = parseInt(document.getElementById('set-flow').value) || 0;
            const p = parseInt(document.getElementById('set-pump').value) || 0;

            if ((h + f + p) !== 100) {
                if(window.Telegram && window.Telegram.WebApp) {
                    window.Telegram.WebApp.showAlert("Матрица нарушена! В сумме должно быть строго 100%. Сейчас: " + (h+f+p) + "%");
                } else {
                    alert("Матрица нарушена! В сумме должно быть строго 100%. Сейчас: " + (h+f+p) + "%");
                }
                return;
            }

            data.settings.hold = h;
            data.settings.flow = f;
            data.settings.pump = p;

            toggleModal('');
            updateUI();
        }

        // Вспомогательная функция для добавления логов в историю
        function addLog(type, amount, desc) {
            if (!data.history) data.history = [];
            data.history.unshift({ type, amount, desc });
            if (data.history.length > 5) data.history.pop(); // Храним только 5 последних логов
        }

        // Первичный запуск отрисовки данных
        updateUI();
    </script>
