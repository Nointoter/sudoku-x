<script lang="ts">
  type CellPosition = { row: number; col: number };
  type BoardState = (number | null)[][];
  type NotesState = Set<number>[][][];
  
  let selectedCell: CellPosition | null = null;
  let board: BoardState = Array(9).fill(null).map(() => Array(9).fill(null));
  let notes: NotesState = Array.from({ length: 9 }, () => 
    Array.from({ length: 9 }, () => 
      Array.from({ length: 9 }, () => new Set())
    )
  );
  
  let errors: boolean[][] = Array(9).fill(null).map(() => Array(9).fill(false));
  let noteMode = false;
  let history: {board: BoardState, notes: NotesState}[] = [];

  let showMenu = false;
  let darkMode = false;
  let backgroundColor = '#ffffff';
  let cellColor = '#ffffff';
  let textColor = '#000000';

  // Добавляем новый тип для режимов игры
  type GameMode = 'blank' | 'classic' | 'killer';
  let gameMode: GameMode = 'blank';
  
  // Функция для генерации классического судоку
  const generateClassicSudoku = () => {
    // Очищаем доску
    board = Array(9).fill(null).map(() => Array(9).fill(null));
    
    // Простая реализация генерации (можно заменить на более сложный алгоритм)
    // Здесь мы просто создаем валидное судоку и затем удаляем часть чисел
    const solution = [
      [5, 3, 4, 6, 7, 8, 9, 1, 2],
      [6, 7, 2, 1, 9, 5, 3, 4, 8],
      [1, 9, 8, 3, 4, 2, 5, 6, 7],
      [8, 5, 9, 7, 6, 1, 4, 2, 3],
      [4, 2, 6, 8, 5, 3, 7, 9, 1],
      [7, 1, 3, 9, 2, 4, 8, 5, 6],
      [9, 6, 1, 5, 3, 7, 2, 8, 4],
      [2, 8, 7, 4, 1, 9, 6, 3, 5],
      [3, 4, 5, 2, 8, 6, 1, 7, 9]
    ];
    
    // Копируем решение и удаляем часть чисел (оставляем ~30-35 чисел)
    const cluesCount = Math.floor(Math.random() * 6) + 30; // 30-35 подсказок
    let cluesAdded = 0;
    
    while (cluesAdded < cluesCount) {
      const row = Math.floor(Math.random() * 9);
      const col = Math.floor(Math.random() * 9);
      
      if (board[row][col] === null) {
        board[row][col] = solution[row][col];
        cluesAdded++;
      }
    }
    
    validateBoard();
  };

  // Обработчик изменения режима игры
  const changeGameMode = (mode: GameMode) => {
    gameMode = mode;
    
    if (mode === 'blank') {
      // Очищаем доску для blank режима
      board = Array(9).fill(null).map(() => Array(9).fill(null));
      notes = Array.from({ length: 9 }, () => 
        Array.from({ length: 9 }, () => 
          Array.from({ length: 9 }, () => new Set())
      ));
      errors = Array(9).fill(null).map(() => Array(9).fill(false));
    } else if (mode === 'classic') {
      generateClassicSudoku();
    } else if (mode === 'killer') {
      // TODO: Реализовать killer режим
      alert('Killer режим будет реализован в будущем');
    }
    
    // Очищаем историю при смене режима
    history = [];
    selectedCell = null;
  };

  // Цветовые схемы
  const colorSchemes = {
    light: {
      background: '#ffffff',
      cell: '#ffffff',
      text: '#000000'
    },
    dark: {
      background: '#1a1a1a',
      cell: '#2d2d2d',
      text: '#e6e6e6'
    },
    sepia: {
      background: '#f4ecd8',
      cell: '#e5d9b6',
      text: '#5b4636'
    }
  };

  // Переключение цветовой схемы
  const setColorScheme = (scheme: keyof typeof colorSchemes) => {
    const colors = colorSchemes[scheme];
    backgroundColor = colors.background;
    cellColor = colors.cell;
    textColor = colors.text;
    darkMode = scheme === 'dark';
  };

  // Закрытие меню при клике вне его
  function handleClickOutside(event: MouseEvent) {
    const menu = document.querySelector('.menu-container');
    const menuButton = document.querySelector('.menu-button');
    if (menu && !menu.contains(event.target as Node) && 
        menuButton && !menuButton.contains(event.target as Node)) {
      showMenu = false;
    }
  }

  // Добавляем обработчик при монтировании компонента
  import { onMount } from 'svelte';
  onMount(() => {
    document.addEventListener('click', handleClickOutside);
    return () => document.removeEventListener('click', handleClickOutside);
  });

  // Сохраняем состояние в историю
  const saveState = () => {
    const boardCopy = board.map(row => [...row]);
    const notesCopy = notes.map(row => 
      row.map(cell => 
        cell.map(set => new Set(set))
      )
    );
    history.push({ board: boardCopy, notes: notesCopy });
    if (history.length > 50) history.shift();
  };

  // Отмена последнего действия
  const undo = () => {
    if (history.length === 0) return;
    
    const prevState = history.pop();
    if (!prevState) return;
    
    board = prevState.board.map(row => [...row]);
    notes = prevState.notes.map(row => 
      row.map(cell => 
        cell.map(set => new Set(set))
      )
    );
    validateBoard();
  };

  // Проверка правил судоку
  const validateBoard = () => {
    errors = Array(9).fill(null).map(() => Array(9).fill(false));
    
    // Проверка строк и столбцов
    for (let i = 0; i < 9; i++) {
      const rowNumbers = new Set<number>();
      const colNumbers = new Set<number>();
      
      for (let j = 0; j < 9; j++) {
        // Проверка строки
        const rowValue = board[i][j];
        if (rowValue !== null) {
          if (rowNumbers.has(rowValue)) {
            for (let k = 0; k < 9; k++) {
              if (board[i][k] === rowValue) {
                errors[i][k] = true;
              }
            }
          }
          rowNumbers.add(rowValue);
        }
        
        // Проверка столбца
        const colValue = board[j][i];
        if (colValue !== null) {
          if (colNumbers.has(colValue)) {
            for (let k = 0; k < 9; k++) {
              if (board[k][i] === colValue) {
                errors[k][i] = true;
              }
            }
          }
          colNumbers.add(colValue);
        }
      }
    }
    
    // Проверка квадратов 3x3
    for (let boxRow = 0; boxRow < 3; boxRow++) {
      for (let boxCol = 0; boxCol < 3; boxCol++) {
        const boxNumbers = new Set<number>();
        const duplicates = new Set<number>();
        
        for (let i = 0; i < 3; i++) {
          for (let j = 0; j < 3; j++) {
            const row = boxRow * 3 + i;
            const col = boxCol * 3 + j;
            const num = board[row][col];
            
            if (num !== null) {
              if (boxNumbers.has(num)) {
                duplicates.add(num);
              }
              boxNumbers.add(num);
            }
          }
        }
        
        if (duplicates.size > 0) {
          for (let i = 0; i < 3; i++) {
            for (let j = 0; j < 3; j++) {
              const row = boxRow * 3 + i;
              const col = boxCol * 3 + j;
              const cellValue = board[row][col];
              if (cellValue !== null && duplicates.has(cellValue)) {
                errors[row][col] = true;
              }
            }
          }
        }
      }
    }
    
    // Удаляем заметки с числами, которые уже есть в строке/столбце/квадрате
    cleanUpNotes();
  };

  const cleanUpNotes = () => {
    for (let row = 0; row < 9; row++) {
      for (let col = 0; col < 9; col++) {
        if (board[row][col] !== null) continue;
        
        // Получаем все числа в текущей строке, столбце и квадрате
        const numbersInRow = getNumbersInRow(row);
        const numbersInCol = getNumbersInCol(col);
        const numbersInBox = getNumbersInBox(row, col);
        
        // Объединяем все запрещенные числа
        const forbiddenNumbers = new Set([
          ...numbersInRow,
          ...numbersInCol,
          ...numbersInBox
        ]);
        
        // Удаляем запрещенные числа из заметок
        const newNotes = [...notes[row][col]];
        for (let num = 1; num <= 9; num++) {
          if (forbiddenNumbers.has(num)) {
            newNotes[num - 1] = new Set();
          }
        }
        notes[row][col] = newNotes;
      }
    }
    notes = [...notes]; // Для реактивности
  };

  const getNumbersInRow = (row: number): number[] => {
    const numbers: number[] = [];
    for (let col = 0; col < 9; col++) {
      if (board[row][col] !== null) {
        numbers.push(board[row][col]!);
      }
    }
    return numbers;
  };

  const getNumbersInCol = (col: number): number[] => {
    const numbers: number[] = [];
    for (let row = 0; row < 9; row++) {
      if (board[row][col] !== null) {
        numbers.push(board[row][col]!);
      }
    }
    return numbers;
  };

  const getNumbersInBox = (row: number, col: number): number[] => {
    const numbers: number[] = [];
    const boxRow = Math.floor(row / 3) * 3;
    const boxCol = Math.floor(col / 3) * 3;
    
    for (let i = 0; i < 3; i++) {
      for (let j = 0; j < 3; j++) {
        const cellValue = board[boxRow + i][boxCol + j];
        if (cellValue !== null) {
          numbers.push(cellValue);
        }
      }
    }
    return numbers;
  };

  // Выбор клетки
  const selectCell = (row: number, col: number) => {
    selectedCell = { row, col };
  };

  // Обработка нажатия клавиш
  const handleKeyDown = (e: KeyboardEvent, row: number, col: number) => {
    const key = e.key;
    
    if (key >= '1' && key <= '9') {
      const num = parseInt(key, 10);
      if (noteMode) {
        toggleNote(row, col, num);
      } else {
        setNumber(num);
      }
    } else if (key === 'Delete' || key === 'Backspace') {
      clearCell();
    } else if (key === 'ArrowUp' && row > 0) {
      selectCell(row - 1, col);
    } else if (key === 'ArrowDown' && row < 8) {
      selectCell(row + 1, col);
    } else if (key === 'ArrowLeft' && col > 0) {
      selectCell(row, col - 1);
    } else if (key === 'ArrowRight' && col < 8) {
      selectCell(row, col + 1);
    } else if (key === 'n' || key === 'N') {
      toggleNoteMode();
    }
  };

  // Ввод цифры
  const setNumber = (num: number) => {
    if (!selectedCell) return;
    const { row, col } = selectedCell;
    
    // В classic режиме нельзя изменять начальные числа
    if (gameMode === 'classic' && board[row][col] !== null) {
      return;
    }
    
    saveState();
    board[row][col] = num;
    notes[row][col] = Array(9).fill(new Set());
    validateBoard();
  };

  // Очистка клетки
  const clearCell = () => {
    if (!selectedCell) return;
    const { row, col } = selectedCell;
    
    // В classic режиме нельзя очищать начальные числа
    if (gameMode === 'classic' && board[row][col] !== null) {
      return;
    }
    
    saveState();
    board[row][col] = null;
    notes[row][col] = Array(9).fill(new Set());
    validateBoard();
  };

  // Переключение заметки
  const toggleNote = (row: number, col: number, num: number) => {
    if (board[row][col] !== null) return;
    
    saveState();
    
    // Создаем новый массив заметок для ячейки
    const newNotes = [...notes[row][col]];
    
    // Создаем новый Set для конкретной цифры
    const newSet = new Set(newNotes[num - 1]); // -1 потому что индексы с 0
    
    if (newSet.has(num)) {
      newSet.delete(num);
    } else {
      newSet.add(num);
    }
    
    newNotes[num - 1] = newSet;
    notes[row][col] = newNotes;
    notes = [...notes]; // Для реактивности
  };

  // Переключение режима заметок
  const toggleNoteMode = () => {
    noteMode = !noteMode;
  };
</script>

<div 
  class="sudoku-container"
  style="
    --bg-color: {backgroundColor};
    --cell-color: {cellColor};
    --text-color: {textColor};
    --primary-color: #{darkMode ? '3a7bd5' : '4a90e2'};
    --darkMode: #{darkMode ? '#3a7bd5' : '#1a5fb4'}
    --error-color: #ff0000;
    --error-cell-bg: #ffcccc;
    --selected-error-bg: #ffb3b3;
    --selected-cell-bg: {darkMode ? '#3a3a3a' : '#d4e6ff'};
    --tool-button-bg: {darkMode ? '#3a3a3a' : '#f0f0f0'};
    --tool-button-hover: {darkMode ? '#4a4a4a' : '#ddd'};
    --control-button-bg: {darkMode ? '#3a3a3a' : '#f0f0f0'};
    --control-button-hover: {darkMode ? '#4a4a4a' : '#ddd'};
    --border-color: {darkMode ? '#555' : '#ccc'};
    --menu-bg: {darkMode ? '#2d2d2d' : '#ffffff'};
    --menu-text: {darkMode ? '#e6e6e6' : '#000000'};
    --notes-color: {darkMode ? '#a0a0a0' : '#666'};
  "
>
  <button 
    class="menu-button"
    on:click|stopPropagation={() => showMenu = !showMenu}
    on:keydown|stopPropagation={(e) => e.key === 'Enter' && (showMenu = !showMenu)}
    aria-label="Открыть меню настроек"
    aria-expanded={showMenu}
    aria-controls="settings-menu"
  >
    <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/>
    </svg>
  </button>

  <!-- Боковое меню -->
  {#if showMenu}
    <!-- svelte-ignore a11y_click_events_have_key_events -->
    <div 
      class="menu-container" 
      on:click|stopPropagation
      role="dialog"
      tabindex="-1"
      aria-modal="true"
      aria-labelledby="settings-title"
      id="settings-menu"
    >
      <h3 id="settings-title">Настройки</h3>
      
      <div class="menu-section">
        <h4>Цветовая схема</h4>
        <div class="color-options">
          <button 
            class="color-option {darkMode ? '' : 'active'}" 
            on:click={() => setColorScheme('light')}
            on:keydown={(e) => e.key === 'Enter' && setColorScheme('light')}
            style="background: #ffffff; color: #000000"
            aria-label="Светлая тема"
          >
            Светлая
          </button>
          <button 
            class="color-option {darkMode ? 'active' : ''}" 
            on:click={() => setColorScheme('dark')}
            on:keydown={(e) => e.key === 'Enter' && setColorScheme('dark')}
            style="background: #2d2d2d; color: #e6e6e6"
            aria-label="Темная тема"
          >
            Темная
          </button>
          <button 
            class="color-option" 
            on:click={() => setColorScheme('sepia')}
            on:keydown={(e) => e.key === 'Enter' && setColorScheme('sepia')}
            style="background: #f4ecd8; color: #5b4636"
            aria-label="Сепия тема"
          >
            Сепия
          </button>
        </div>
      </div>
      
      <div class="menu-section">
        <h4>Настроить цвета</h4>
        <div class="color-picker">
          <label>
            Фон:
            <input type="color" bind:value={backgroundColor} aria-label="Выбор цвета фона">
          </label>
          <label>
            Клетки:
            <input type="color" bind:value={cellColor} aria-label="Выбор цвета клеток">
          </label>
          <label>
            Текст:
            <input type="color" bind:value={textColor} aria-label="Выбор цвета текста">
          </label>
        </div>
      </div>
    </div>
    
    <!-- Затемнение фона -->
    <div class="menu-overlay" role="presentation"></div>
  {/if}


  <!-- Остальная часть интерфейса (h1, sudoku-grid, toolbar, controls) -->
  <h1>Sudoku-X</h1>

  <div class="mode-selector">
    <button 
      class:active={gameMode === 'blank'}
      on:click={() => changeGameMode('blank')}
      aria-label="Blank режим - полностью редактируемое поле"
    >
      Blank
    </button>
    <button 
      class:active={gameMode === 'classic'}
      on:click={() => changeGameMode('classic')}
      aria-label="Classic режим - классическое судоку с подсказками"
    >
      Classic
    </button>
    <button 
      class:active={gameMode === 'killer'}
      on:click={() => changeGameMode('killer')}
      aria-label="Killer режим - судоку с дополнительными правилами"
    >
      Killer
    </button>
  </div>

  <!-- Игровое поле 9x9 -->
  <div class="sudoku-grid">
    {#each board as row, rowIndex}
      {#each row as cell, colIndex}
        <button
          class="cell"
          class:initial={gameMode === 'classic' && cell !== null}
          class:selected={selectedCell?.row === rowIndex && selectedCell?.col === colIndex}
          class:error={errors[rowIndex][colIndex]}
          on:click={() => selectCell(rowIndex, colIndex)}
          on:keydown={(e) => handleKeyDown(e, rowIndex, colIndex)}
          aria-label="{cell ? `Ячейка ${rowIndex+1}-${colIndex+1}, значение ${cell}` : `Пустая ячейка ${rowIndex+1}-${colIndex+1}`}"
          tabindex="0"
          style="background-color: {cellColor}"
        >
          {#if cell}
            {cell}
          {:else}
            <div class="notes-grid">
              {#each [1, 2, 3, 4, 5, 6, 7, 8, 9] as num}
                {#if notes[rowIndex][colIndex][num - 1].size > 0}
                  <div class="note">{num}</div>
                {:else}
                  <div class="note empty"></div>
                {/if}
              {/each}
            </div>
          {/if}
        </button>
      {/each}
    {/each}
  </div>

  <!-- Панель инструментов -->
  <div class="toolbar">
    <button on:click={undo} class="tool-button" title="Отменить (Ctrl+Z)" aria-label="Отменить последнее действие">
      <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true">
        <path d="M12.5 8c-2.65 0-5.05.99-6.9 2.6L2 7v9h9l-3.62-3.62c1.39-1.16 3.16-1.88 5.12-1.88 3.54 0 6.55 2.31 7.6 5.5l2.37-.78C21.08 11.03 17.15 8 12.5 8z"/>
      </svg>
    </button>
    <button on:click={clearCell} class="tool-button" title="Очистить ячейку (Del)" aria-label="Очистить текущую ячейку">
      <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true">
        <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
      </svg>
    </button>
    <button 
      on:click={toggleNoteMode} 
      class="tool-button" 
      class:active={noteMode}
      title="Режим заметок (N)"
      aria-label="{noteMode ? 'Выключить режим заметок' : 'Включить режим заметок'}"
    >
      <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true">
        <path d="M18 2H6c-1.1 0-2 .9-2 2v16c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2zM6 4h5v8l-2.5-1.5L6 12V4z"/>
      </svg>
    </button>
  </div>

  <!-- Панель цифр -->
  <div class="controls">
    {#each [1, 2, 3, 4, 5, 6, 7, 8, 9] as num}
      <button 
        on:click={() => {
          if (selectedCell) {
            noteMode 
              ? toggleNote(selectedCell.row, selectedCell.col, num) 
              : setNumber(num);
          }
        }}
        aria-label="{noteMode ? `Добавить заметку ${num}` : `Установить значение ${num}`}"
      >
        {num}
      </button>
    {/each}
  </div>
</div>

<style lang="postcss">
  .sudoku-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
    padding: 1rem;
    font-family: Arial, sans-serif;
    max-width: 600px;
    margin: 0 auto;
    background-color: var(--bg-color);
    color: var(--text-color);
  }

  .toolbar {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 0.5rem;
    width: 100%;
    justify-content: center;
  }

  .tool-button {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border: 1px solid var(--border-color);
    background: var(--tool-button-bg);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    transition: all 0.2s;
    color: var(--text-color);
  }

  .tool-button:hover {
    background: var(--tool-button-hover);
  }

  .tool-button.active {
    background: var(--primary-color);
    color: white;
    border-color: var(--primary-color);
  }

  .sudoku-grid {
    display: grid;
    grid-template-columns: repeat(9, minmax(30px, 1fr));
    grid-template-rows: repeat(9, minmax(30px, 1fr));
    aspect-ratio: 1/1;
    width: 100%;
    max-width: 500px;
    gap: 1px;
    background: #333;
    border: 2px solid #333;
  }

  .cell {
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--cell-color);
    font-size: 1.5rem;
    cursor: pointer;
    user-select: none;
    border: none;
    padding: 0;
    margin: 0;
    outline: none;
    position: relative;
    aspect-ratio: 1/1;
    color: var(--text-color);
  }

  .notes-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
    width: 100%;
    height: 100%;
    font-size: 0.7rem;
    color: var(--notes-color);
  }

  .note {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .note.empty {
    opacity: 0;
  }

  .cell.selected {
    background: var(--selected-cell-bg);
    box-shadow: inset 0 0 0 2px var(--primary-color);
  }

  .cell.error {
    background: var(--error-cell-bg);
    color: var(--error-color);
  }

  .cell.selected.error {
    background: var(--selected-error-bg);
    box-shadow: inset 0 0 0 2px var(--error-color);
  }

  /* Границы для блоков 3x3 */
  .cell:nth-child(3n) {
    border-right: 2px solid #333;
  }
  .cell:nth-child(9n) {
    border-right: none;
  }
  .cell:nth-child(n + 19):nth-child(-n + 27),
  .cell:nth-child(n + 46):nth-child(-n + 54) {
    border-bottom: 2px solid #333;
  }

  .controls {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 0.5rem;
    width: 100%;
    max-width: 500px;
  }

  .controls button {
    aspect-ratio: 1/1;
    font-size: 1.2rem;
    border: 1px solid var(--border-color);
    background: var(--control-button-bg);
    cursor: pointer;
    border-radius: 4px;
    transition: background 0.2s;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    color: var(--text-color);
  }

  .controls button:hover {
    background: var(--control-button-hover);
  }

  .menu-button {
    position: fixed;
    top: 20px;
    right: 20px;
    background: none;
    border: none;
    cursor: pointer;
    padding: 8px;
    z-index: 100;
    color: var(--text-color);
  }
  
  .menu-container {
    position: fixed;
    top: 0;
    right: 0;
    width: 300px;
    height: 100vh;
    background-color: var(--menu-bg);
    color: var(--menu-text);
    box-shadow: -2px 0 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    z-index: 1000;
    overflow-y: auto;
  }
  
  .menu-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 999;
  }
  
  .menu-section {
    margin-bottom: 20px;
  }
  
  .color-options {
    display: flex;
    gap: 10px;
    margin-top: 10px;
  }
  
  .color-option {
    padding: 8px 12px;
    border-radius: 4px;
    cursor: pointer;
    border: 1px solid var(--border-color);
    transition: all 0.2s;
  }
  
  .color-option.active {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 2px rgba(74, 144, 226, 0.3);
  }
  
  .color-picker {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }
  
  .color-picker label {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  @media (max-width: 500px) {
    .sudoku-grid {
      max-width: 100vw;
    }
    
    .controls {
      grid-template-columns: repeat(5, 1fr);
    }
  }

  .mode-selector {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 1rem;
  }

  .mode-selector button {
    padding: 0.5rem 1rem;
    border: 1px solid var(--border-color);
    background: var(--control-button-bg);
    color: var(--text-color);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .mode-selector button:hover {
    background: var(--control-button-hover);
  }

  .mode-selector button.active {
    background: var(--primary-color);
    color: white;
    border-color: var(--primary-color);
  }

  /* Стиль для начальных чисел в classic режиме */
  .cell.initial {
    font-weight: bold;
    color: var(--darkMode);
  }
</style>