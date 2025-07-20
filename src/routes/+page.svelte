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
    
    saveState();
    const { row, col } = selectedCell;
    board[row][col] = num;
    // Очищаем заметки для этой клетки
    notes[row][col] = Array(9).fill(new Set());
    validateBoard();
  };

  // Очистка клетки
  const clearCell = () => {
    if (!selectedCell) return;
    
    saveState();
    const { row, col } = selectedCell;
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

<div class="sudoku-container">
  <h1>Sudoku-X</h1>

  <!-- Игровое поле 9x9 -->
  <div class="sudoku-grid">
    {#each board as row, rowIndex}
      {#each row as cell, colIndex}
        <button
          class="cell"
          class:selected={selectedCell?.row === rowIndex && selectedCell?.col === colIndex}
          class:error={errors[rowIndex][colIndex]}
          on:click={() => selectCell(rowIndex, colIndex)}
          on:keydown={(e) => handleKeyDown(e, rowIndex, colIndex)}
          aria-label="{cell ? `Ячейка ${rowIndex+1}-${colIndex+1}, значение ${cell}` : `Пустая ячейка ${rowIndex+1}-${colIndex+1}`}"
          tabindex="0"
        >
          {#if cell}
            {cell}
          {:else}
            <div class="notes-grid">
              {#each [1, 2, 3, 4, 5, 6, 7, 8, 9] as num}
                {#if notes[rowIndex][colIndex][num - 1].size > 0} <!-- -1 потому что индексы с 0 -->
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

<style>
  .sudoku-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
    padding: 1rem;
    font-family: Arial, sans-serif;
    max-width: 600px;
    margin: 0 auto;
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
    border: 1px solid #ccc;
    background: #f0f0f0;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    transition: all 0.2s;
  }

  .tool-button:hover {
    background: #ddd;
  }

  .tool-button.active {
    background: #4a90e2;
    color: white;
    border-color: #4a90e2;
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
    background: white;
    font-size: 1.5rem;
    cursor: pointer;
    user-select: none;
    border: none;
    padding: 0;
    margin: 0;
    outline: none;
    position: relative;
    aspect-ratio: 1/1;
  }

  .notes-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
    width: 100%;
    height: 100%;
    font-size: 0.7rem;
    color: #666;
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
    background: #d4e6ff;
    box-shadow: inset 0 0 0 2px #4a90e2;
  }

  .cell.error {
    background: #ffcccc;
    color: #ff0000;
  }

  .cell.selected.error {
    background: #ffb3b3;
    box-shadow: inset 0 0 0 2px #ff0000;
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
    border: 1px solid #ccc;
    background: #f0f0f0;
    cursor: pointer;
    border-radius: 4px;
    transition: background 0.2s;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
  }

  .controls button:hover {
    background: #ddd;
  }

  @media (max-width: 500px) {
    .sudoku-grid {
      max-width: 100vw;
    }
    
    .controls {
      grid-template-columns: repeat(5, 1fr);
    }
  }
</style>