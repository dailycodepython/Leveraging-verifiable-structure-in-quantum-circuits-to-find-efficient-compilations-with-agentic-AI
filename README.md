# Leveraging-verifiable-structure-in-quantum-circuits-to-find-efficient-compilations-with-agentic-AI

# Google Quantum AI: Verifiable Quantum Circuit Compilation (Kickmix)
Этот ноутбук содержит теоретический разбор, архитектурный конспект и программную симуляцию компилятора квантовых схем **Kickmix** по мотивам свежего исследования Google Quantum AI (*arXiv:2610.23832*).

---

## 📖 1. Теоретический конспект (Paper Review)

### В чем суть прорыва Kickmix?
Главная проблема квантовой компиляции (оптимизации схем) — **экспоненциальная сложность верификации**. Если вы случайно измените логику схемы при сокращении вентилей, то для проверки эквивалентности классическим путем придется перемножать огромные матрицы, что физически невозможно для больших систем.

Исследователи из Google Quantum AI выделили особый класс схем — **Kickmix-схемы**. Их ключевые свойства:
* **Сохранение базиса:** Они переводят каждый входной классический битстринг $|x\rangle$ строго в один выходной битстринг $|y\rangle$.
* **Отсутствие суперпозиции:** В процессе вычислений над классическими данными внутри схемы *не создается квантовая суперпозиция*. Вместо этого на состояния накладывается определенная квантовая фаза $e^{i\phi(x)}$.

**Почему это важно для Kaggle?** Это свойство позволяет проводить **классический фаззинг (fuzz testing)**. Мы можем прогнать через исходную и оптимизированную схему случайные битовые строки на обычном процессоре и мгновенно проверить, совпадает ли результат и фаза. Скорость такой проверки достигает **60 миллиардов вентилей в секунду**, что делает задачу идеальной для поиска решений с помощью ИИ-агентов (RL, MCTS или LLM).

### Архитектура Kickmix-схемы
Каждая схема в турнире состоит из трех строго определенных слоев:
1. **Обратимая классическая логика (Reversible Logic):** Вентили Тоффоли (Toffoli), CNOT и NOT. Они вычисляют промежуточные функции и адреса для QROM.
2. **Диагональные фазовые вентили (Diagonal Phase Gates):** «Пинают» (kick) состояние кубитов, накладывая фазу, зависящую от состояния классического регистра. Отсюда название *Kick*.
3. **Развычисление на основе измерений (Measurement-based Uncomputation / HMR):** Очистка «мусорных» кубитов и анцилл через измерения и классический сдвиг фазы (feed-forward).

## 📊 2. Граф вычислений Kickmix

Посмотрите, как устроена сквозная компиляция и прохождение информации через квантовые регистры:
'''
import matplotlib.pyplot as plt
import matplotlib.patches as patches

def draw_kickmix_pipeline():
    # Настройки фигуры
    fig, ax = plt.subplots(figsize=(10, 8), dpi=120)
    ax.set_xlim(0, 10)
    ax.set_ylim(0, 12)
    ax.axis('off')
    
    # Стили текста и блоков
    box_style = dict(boxstyle="round,pad=0.6", ec="#2c3e50", lw=2)
    font_main = dict(fontsize=12, fontweight='bold', color='#ffffff')
    font_desc = dict(fontsize=11, color='#2c3e50', style='italic')
    
    # 1. Входные данные
    ax.text(5, 11, r"$\mathbf{[ \, Классический \,\, Вход: \,\, |x\rangle \, ]}$", 
            ha='center', va='center', fontsize=12, fontweight='bold',
            bbox=dict(boxstyle="square,pad=0.4", fc="#e1f5fe", ec="#0288d1", lw=1.5))
    
    # Стрелка от входа к Слою 1
    ax.annotate('', xy=(5, 9.4), xytext=(5, 10.5),
                arrowprops=dict(arrowstyle="-|>", color="#2c3e50", lw=2, mutation_scale=15))
    
    # 2. Слой 1
    rect1 = patches.FancyBboxPatch((1.5, 7.8), 7, 1.4, boxstyle="round,pad=0.1", 
                                   fc="#2980b9", ec="#1f3a52", lw=2)
    ax.add_patch(rect1)
    ax.text(5, 8.5, "Layer 1: Reversible Logic", ha='center', va='center', **font_main)
    ax.text(5, 7.3, "──►  Вычисляет адреса и промежуточные биты (Toffoli, CNOT)", ha='center', va='center', **font_desc)
    
    # Стрелка от Слой 1 к Слою 2
    ax.annotate('', xy=(5, 5.8), xytext=(5, 6.9),
                arrowprops=dict(arrowstyle="-|>", color="#2c3e50", lw=2, mutation_scale=15))
    
    # 3. Слой 2
    rect2 = patches.FancyBboxPatch((1.5, 4.2), 7, 1.4, boxstyle="round,pad=0.1", 
                                   fc="#27ae60", ec="#145a32", lw=2)
    ax.add_patch(rect2)
    ax.text(5, 4.9, "Layer 2: Diagonal Phase (Kick)", ha='center', va='center', **font_main)
    ax.text(5, 3.7, r"──►  Накладывает квантовую фазу $e^{i\phi(x)}$ без суперпозиции", ha='center', va='center', **font_desc)
    
    # Стрелка от Слой 2 к Слою 3
    ax.annotate('', xy=(5, 2.2), xytext=(5, 3.3),
                arrowprops=dict(arrowstyle="-|>", color="#2c3e50", lw=2, mutation_scale=15))
    
    # 4. Слой 3
    rect3 = patches.FancyBboxPatch((1.5, 0.6), 7, 1.4, boxstyle="round,pad=0.1", 
                                   fc="#e67e22", ec="#6e2c00", lw=2)
    ax.add_patch(rect3)
    ax.text(5, 1.3, "Layer 3: HMR Uncomputation", ha='center', va='center', **font_main)
    ax.text(5, 0.1, "──►  Измеряет анциллы + классически правит фазу (feed-forward)", ha='center', va='center', **font_desc)
    
    # Стрелка к выходу
    ax.annotate('', xy=(5, -1.2), xytext=(5, -0.3),
                arrowprops=dict(arrowstyle="-|>", color="#2c3e50", lw=2, mutation_scale=15))
    
    # 5. Выходные данные
    ax.text(5, -1.8, r"$\mathbf{[ \, Классический \,\, Выход: \,\, |y\rangle + \Phi а з а \, ]}$", 
            ha='center', va='center', fontsize=12, fontweight='bold',
            bbox=dict(boxstyle="square,pad=0.4", fc="#fff3e0", ec="#f57c00", lw=1.5))

    plt.tight_layout()
    plt.show()

# Запуск отрисовки схемы
draw_kickmix_pipeline()
'''

## 💻 3. Программная симуляция Kickmix-компилятора

Скрипт ниже моделирует упрощенную структуру Kickmix-компиляции. Он принимает логическое выражение, строит базовую схему, а затем применяет **эвристический шаг оптимизации (схлопывание фаз)**, проверяя эквивалентность через классический фаззинг (как на турнире Google).
'''
import random
import math

class KickmixCircuit:
    def __init__(self, num_qubits):
        self.num_qubits = num_qubits
        # Схема хранится как последовательность вентилей: (тип, целевые_кубиты, параметры)
        self.gates = [] 

    def add_toffoli(self, ctrl1, ctrl2, target):
        self.gates.append(('TOFFOLI', (ctrl1, ctrl2), target))

    def add_phase_kick(self, target_qubit, angle):
        self.gates.append(('PHASE_KICK', target_qubit, angle))

    def add_hmr_uncompute(self, qubit):
        self.gates.append(('HMR', qubit, None))

    def compute_classical(self, input_bitstring):
        """
        Классический эмулятор (Fuzzer) для Kickmix структуры.
        Возвращает: (выходной_битстринг, итоговая_фаза)
        """
        state = list(input_bitstring)
        phase = 0.0
        
        for gate_type, qubits, param in self.gates:
            if gate_type == 'TOFFOLI':
                c1, c2 = qubits
                target = param
                if state[c1] == 1 and state[c2] == 1:
                    state[target] = 1 - state[target]
            
            elif gate_type == 'PHASE_KICK':
                target = qubits
                angle = param
                # Фаза накладывается только если кубит в состоянии |1>
                if state[target] == 1:
                    phase += angle
            
            elif gate_type == 'HMR':
                # В kickmix измерение анциллы в классическом симуляторе 
                # сбрасывает её без разрушения фазы основного регистра
                target = qubits
                state[target] = 0 
                
        return tuple(state), phase % (2 * math.pi)

class KickmixOptimizer:
    """ Упрощенный компилятор для поиска сокращений в схеме """
    @staticmethod
    def optimize_phase_gadgets(circuit):
        optimized = KickmixCircuit(circuit.num_qubits)
        i = 0
        while i < len(circuit.gates):
            # Эвристика: если идут два фазовых пинка подряд на один и тот же кубит, схлопываем их фазы
            if (i < len(circuit.gates) - 1 and 
                circuit.gates[i][0] == 'PHASE_KICK' and 
                circuit.gates[i+1][0] == 'PHASE_KICK' and 
                circuit.gates[i][1] == circuit.gates[i+1][1]):
                
                q = circuit.gates[i][1]
                combined_angle = circuit.gates[i][2] + circuit.gates[i+1][2]
                optimized.add_phase_kick(q, combined_angle)
                i += 2
            else:
                # Копируем вентиль без изменений
                g_type, q, p = circuit.gates[i]
                if g_type == 'TOFFOLI':
                    optimized.add_toffoli(q[0], q[1], p)
                elif g_type == 'PHASE_KICK':
                    optimized.add_phase_kick(q, p)
                elif g_type == 'HMR':
                    optimized.add_hmr_uncompute(q)
                i += 1
        return optimized

    @staticmethod
    def verify_equivalence(circuit_a, circuit_b, num_tests=500):
        """ Турнирный Фаззинг (Fuzz Testing) для проверки корректности компиляции """
        n = circuit_a.num_qubits
        for _ in range(num_tests):
            # Генерируем случайный входной вектор битов
            test_input = tuple(random.randint(0, 1) for _ in range(n))
            
            res_a, phase_a = circuit_a.compute_classical(test_input)
            res_b, phase_b = circuit_b.compute_classical(test_input)
            
            # Проверяем строгое совпадение конечного состояния и квантовой фазы
            if res_a != res_b or not math.isclose(phase_a, phase_b, abs_tol=1e-5):
                return False
        return True
'''
'''
# --- Запуск симуляции ---
# Создаем квантовый регистр на 4 кубита
circuit = KickmixCircuit(num_qubits=4)

# Строим избыточную схему (загрузка данных + фаза + избыточная фаза + развычисление)
circuit.add_toffoli(0, 1, 2)
circuit.add_phase_kick(2, math.pi / 4)
circuit.add_phase_kick(2, math.pi / 4) # Дублирующий вентиль фазы
circuit.add_hmr_uncompute(2)

print(f"Исходное количество гейтов в схеме: {len(circuit.gates)}")

# Запускаем оптимизацию
optimizer = KickmixOptimizer()
optimized_circuit = optimizer.optimize_phase_gadgets(circuit)

print(f"Количество гейтов после работы оптимизатора: {len(optimized_circuit.gates)}")

# Турнирная верификация (Фаззинг)
is_valid = optimizer.verify_equivalence(circuit, optimized_circuit)
print(f"\nРезультат верификации: {'УСПЕШНО (Схемы эквивалентны)' if is_valid else 'ОШИБКА (Нарушена логика!)'}")

'''
## 📈 4. Ключевые выводы и Инсайды для соревнования

На основе анализа препринта Google, для достижения топовых мест на Kaggle необходимо сфокусироваться на трех ключевых направлениях:

* **Сокращение T-глубины через правила коммутации:** Слой диагональных фаз (`Phase Kicks`) можно безболезненно переставлять местами и объединять. Чем меньше глубина фазового слоя, тем выше ваш итоговый скор.
* **Оптимизация менеджера анцилл (QROM):** Большинство тяжелых вентилей Toffoli тратится на генерацию условных переходов классической логики. Использование алгоритмов синтеза обратимых схем (например, ESOP-синтез) позволяет кардинально снизить число гейтов первого слоя.
* **Построение поискового Агента (MCTS / RL):** Поскольку классический фаззинг работает мгновенно, лучшей стратегией является построение дерева поиска (Monte Carlo Tree Search), где действиями агента являются локальные правила рерайтинга схемы, а наградой — уменьшение общего веса получившейся структуры.

# Leveraging Verifiable Structure in Quantum Circuits to Find Efficient Compilations with Agentic AI

Реализация и разбор архитектуры компиляции квантовых схем **Kickmix** по мотивам исследования **Google Quantum AI** (*arXiv:2610.23832*).

---

