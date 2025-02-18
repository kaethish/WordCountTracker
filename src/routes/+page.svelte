<script>
    import { onMount } from 'svelte';
  
    /* ------------------- STATE VARIABLES ------------------- */
  
    // Word Goal (v1/v2/v3) – goalMode: "input" (v1), "display" (v2) or "edit" (v3)
    let goalMode = 'input';
    let wordGoal = 0;
    let baselineCount = 0; // record of word total at goal set
    let currentProgress = 0;
    let goalInput = '';
  
    // Logs array: each entry has time, words, rating and an "editing" flag
    let logs = [];
  
    // Log form state
    let logCount = '';
    let logRating = 0;
    let hoverRating = 0;
  
    // Reminder timer state
    const reminderOptions = [
      { label: '10m', seconds: 600 },
      { label: '15m', seconds: 900 },
      { label: '30m', seconds: 1800 },
      { label: '45m', seconds: 2700 },
      { label: '1hr', seconds: 3600 }
    ];
    let selectedReminder = null;
    let timerRemaining = 0;
    let timerInterval = null;
    let timerActive = false;
    let timerPaused = false;
  
    // Confirmation prompt overlay state
    let showPrompt = false;
    let promptMessage = '';
    let promptCallback = null;
  
    /* ------------------- FUNCTIONS ------------------- */
  
    // Request notification permission on mount
    onMount(() => {
      if (Notification && Notification.permission !== "granted") {
        Notification.requestPermission();
      }
    });
  
    // --- Goal / Word Count Tracker functions ---
    function setGoal() {
      const goalVal = parseInt(goalInput);
      if (!isNaN(goalVal) && goalVal > 0) {
        wordGoal = goalVal;
        // Record the baseline so only logs added after setting count for this goal count
        baselineCount = currentProgress;
        goalMode = 'display';
        goalInput = '';
      }
    }
  
    function editGoal() {
      goalMode = 'edit';
      // Pre-fill the input with the current goal value
      goalInput = wordGoal;
    }
  
    function updateGoal() {
      const newGoal = parseInt(goalInput);
      if (!isNaN(newGoal) && newGoal > 0) {
        wordGoal = newGoal;
        // Reset baseline so new logs count toward this new goal only
        baselineCount = currentProgress;
        goalMode = 'display';
        goalInput = '';
      }
    }
  
    function resetGoal() {
      openPrompt("Are you sure you want to reset your goal?", () => {
        // Reset goal progress (but do not erase the logs)
        wordGoal = 0;
        // Reset baseline so that progress becomes zero (currentProgress-baseline becomes 0)
        baselineCount = currentProgress;
        currentProgress = 0;
        goalMode = 'input';
      });
    }
  
    // --- Reminder Timer functions ---
    function startReminder(seconds) {
      selectedReminder = seconds;
      timerRemaining = seconds;
      timerActive = true;
      timerPaused = false;
      if (timerInterval) clearInterval(timerInterval);
      timerInterval = setInterval(() => {
        if (!timerPaused) {
          timerRemaining--;
          if (timerRemaining <= 0) {
            clearInterval(timerInterval);
            timerActive = false;
            timerPaused = false;
            // Trigger browser notification if allowed
            if (Notification.permission === "granted") {
              new Notification("Reminder", { body: "Time's up!" });
            }
          }
        }
      }, 1000);
    }
  
    function pauseTimer() {
      timerPaused = !timerPaused;
    }
  
    function stopTimer() {
      clearInterval(timerInterval);
      timerActive = false;
      timerPaused = false;
      timerRemaining = 0;
      selectedReminder = null;
    }
  
    // --- Log Form and Table functions ---
    function setLogRating(r) {
      logRating = r;
    }
    
    function setHoverRating(q) {
      hoverRating = q;
    }
  
  function logEntry() {
    const count = parseInt(logCount);
    if (!isNaN(count)) {
      const now = new Date();
      const timeStr =
        now.getHours().toString().padStart(2, '0') +
        ":" +
        now.getMinutes().toString().padStart(2, '0');
      logs = [
        ...logs,
        {
          id: Date.now(),
          time: timeStr,
          words: count,
          rating: logRating,
          editing: false,
          // For editing mode, store temporary values:
          editCount: count,
          editTime: timeStr,
          editRating: logRating
        }
      ];
      // Only add to the goal progress if a goal is set.
      if (wordGoal > 0) {
        currentProgress += count;
      }
      // Reset log form
      logCount = '';
      logRating = 0;
    }
  }
  
  function editLog(log) {
    logs = logs.map(l =>
      l.id === log.id
      ? { ...l, editing: true, editCount: l.words, editTime: l.time, editRating: l.rating }
      : l
  );
}

  
function saveLog(log) {
  const oldCount = log.words;
  logs = logs.map(l =>
    l.id === log.id
      ? { ...l, 
          words: parseInt(l.editCount),
          time: l.editTime,
          rating: l.editRating,
          editing: false
        }
      : l
  );
  if (wordGoal > 0) {
    const newLog = logs.find(l => l.id === log.id);
    currentProgress = currentProgress - oldCount + newLog.words;
  }
}
  
  function deleteLog(index) {
    const removed = logs.splice(index, 1)[0];
    // Update progress if a goal is set
    if (wordGoal > 0) {
      currentProgress -= removed.words;
    }
    // Trigger reactivity for logs array
    logs = [...logs];
  }

  function resetLogs() {
    openPrompt("Are you sure you want to reset your word count logs?", () => {
      logs = [];
      currentProgress = 0;
    });
  }
  
    // --- Prompt Overlay functions ---
  function openPrompt(message, callback) {
    promptMessage = message;
    promptCallback = callback;
    showPrompt = true;
  }
  
  function handlePromptResponse(response) {
    showPrompt = false;
    if (response && promptCallback) {
      promptCallback();
    }
  }
  </script>
  
  <!-- MAIN MARKUP -->
  <div class="container">
    <!-- Section I: Logo -->
    <div class="logo">
      <div>Word Count</div>
      <div>Tracker</div>
    </div>
  
    <!-- Section II: Main Content -->
    <div class="main">
      <!-- LEFT SECTION -->
      <div class="left">
        <!-- Top II.a.1: Goal Tracker Component -->
        {#if goalMode === 'input'}
          <!-- v1: Input + SET GOAL -->
          <div class="goal-input v1">
            <div class="goal-container">
              <div class="goal-input-box">
                <input type="number" bind:value={goalInput} />
                <span class="fixed-text">words</span>
              </div>
              <button on:click={setGoal}>SET GOAL</button>
            </div>
          </div>
        {:else if goalMode === 'display'}
          <!-- v2: Display progress with edit icon -->
          <div class="goal-display v2">
            <span class="goal-text">{currentProgress}/{wordGoal} words</span>
            <button on:click={editGoal} class="edit-icon">✎</button>
          </div>
        {:else if goalMode === 'edit'}
          <!-- v3: Edit mode: input with two buttons -->
          <div class="goal-edit v3">
            <div class="goal-container">
              <div class="goal-input-box">
                <input type="number" bind:value={goalInput} />
                <span class="fixed-text">words</span>
              </div>
              <div class="edit-buttons">
                <button on:click={updateGoal}>EDIT</button>
                <button on:click={resetGoal}>RESET</button>
              </div>
            </div>
          </div>
        {/if}
  
        <!-- Bottom II.a.2: Reminder and Log Form -->
        <div class="reminder-log">
          <!-- Reminder (II.a.2.x) -->
          <div class="reminder">
            <div class="reminder-top">
              <div class="reminder-text">Reminder:</div>
              {#if timerActive}
                <div class="timer-display">
                  {Math.floor(timerRemaining / 60)}:{(timerRemaining % 60).toString().padStart(2, '0')}
                  <button on:click={pauseTimer} class="icon-btn">{timerPaused ? "▶" : "⏸"}</button>
                  <button on:click={stopTimer} class="icon-btn">⏹</button>
                </div>
              {/if}
            </div>
            <div class="reminder-options {timerActive ? 'disabled' : ''}">
              {#each reminderOptions as option}
                <div class="reminder-option {selectedReminder === option.seconds ? 'selected' : ''}"
                     on:click={() => { if (!timerActive) startReminder(option.seconds); }}>
                  {option.label}
                </div>
              {/each}
            </div>
          </div>
  
          <!-- Log Form (II.a.2.y) -->
          <div class="log-form">
            <div class="log-top">
              <div>Word Count:</div>
              <input type="number" bind:value={logCount} />
            </div>
            <div class="log-bottom">
              <div class="rating">
                <div class="rating-text">Rating:</div>
                <div class="starsEtButton">
                  <div class="stars">
                    {#each [1, 2, 3, 4, 5] as star}
                      <svg on:click={() => setLogRating(star)} on:mouseenter={() => setHoverRating(star)} on:mouseleave={() => setHoverRating(0)} class="star" width="30" height="30" viewBox="0 0 24 24">
                        <polygon class="stargon" points="12,2 15,9 22,9 17,14 19,21 12,17 5,21 7,14 2,9 9,9"
                                fill={star <= logRating ? "#66FFED" : star <= hoverRating ? "#66FFED" : 'none'}
                                
                                stroke="#66FFED" stroke-width="2" />
                      </svg>
                    {/each}
                  </div>
                  <button on:click={logEntry}>LOG</button>
              </div>
              
            </div>
            </div>
          </div>
        </div>
      </div>
  
      <!-- RIGHT SECTION: Logs Table -->
      <div class="right">
        {#if logs.length > 0}
          <div class="table-reset">
            <button on:click={resetLogs} class="reset-icon">↺</button>
          </div>
        {/if}
        <div class="table-wrapper">
        <table class:emptyTable={logs.length === 0}>
          <thead>
            <tr>
              <th>Time</th>
              <th>Words</th>
              <th>Rating</th>
            </tr>
          </thead>
          <tbody>
            {#each logs as log, index}
              <tr>
                <td>
                  {#if log.editing}
                    <input type="text" bind:value={log.editTime} class="edit-input" />
                  {:else}
                    {log.time}
                  {/if}
                  {#if !log.editing}
                    <button on:click={() => editLog(log)} class="table-icon">✎</button>
                  {/if}
                  {#if log.editing}
                    <button on:click={() => deleteLog(index)} class="table-icon">🗑</button>
                  {/if}
                </td>
                <td>
                  {#if log.editing}
                    <input type="number" bind:value={log.editCount} class="edit-input" />
                  {:else}
                    {log.words}
                  {/if}
                </td>
                <td>
                  {#if log.editing}
                    <div class="logStars edit-stars">
                      {#each [1, 2, 3, 4, 5] as star}
                        <svg on:click={() => log.editRating = star} class="logStar" width="20" height="20" viewBox="0 0 24 24">
                          <polygon points="12,2 15,9 22,9 17,14 19,21 12,17 5,21 7,14 2,9 9,9"
                                   fill={star <= log.editRating ? "#66FFED" : "none"}
                                   stroke="#66FFED" stroke-width="4" />
                        </svg>
                      {/each}
                    </div>
                  {:else}
                    <div class="logStars">
                      {#each [1, 2, 3, 4, 5] as star}
                        <svg class="logStar" width="20" height="20" viewBox="0 0 24 24">
                          <polygon points="12,2 15,9 22,9 17,14 19,21 12,17 5,21 7,14 2,9 9,9"
                                   fill={star <= log.rating ? "#66FFED" : "none"}
                                   stroke="#66FFED" stroke-width="4" />
                        </svg>
                      {/each}
                    </div>
                  {/if}
                </td>
                {#if log.editing}
                  <td>
                    <button on:click={() => saveLog(log)} class="save-btn">SAVE </button>
                  </td>
                {/if}
              </tr>
            {/each}
          </tbody>
        </table>
        </div>
      </div>
    </div>
  </div>
  
  <!-- Confirmation Prompt Overlay -->
  {#if showPrompt}
    <div class="overlay">
      <div class="prompt">
        <div class="prompt-text">{promptMessage}</div>
        <div class="prompt-buttons">
          <button on:click={() => handlePromptResponse(false)}>NO</button>
          <button on:click={() => handlePromptResponse(true)}>YES</button>
        </div>
      </div>
    </div>
  {/if}
  
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Reenie+Beanie&family=Road+Rage&display=swap');
  
    /* Container & Global Styles */
    .container {
      background-color: #66FFED;
      min-height: 90vh;
      padding: 20px;
    }
    .logo {
      text-align: center;
      font-family: 'Reenie Beanie', cursive;
      font-size: 4rem;
      color: black;
      margin-bottom: 20px;
      margin-bottom: 4rem;
    }
    .logo div { line-height: 1.2; }
    .main {
      display: flex;
      justify-content: center;
      gap: 20rem;
      font-family: 'Road Rage', cursive;
    }
    .left, .right { background: transparent; }
    .left {
      display: flex;
      flex-direction: column;
      gap: 3rem;
      width: 400px;
    }
  
    /* --- Goal Tracker (Top II.a.1) --- */
    .goal-container {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      transform: skew(1deg, -2deg);
    }

    .goal-input-box {
      background-color: #66FFED;
      border: 1px solid black;
      border-top-left-radius: 8.44px;
      border-top-right-radius: 8.44px;
      border-bottom-right-radius: 8.44px;
      border-bottom-left-radius: 0;
      color: black;
      padding: 10px;
      display: flex;
      align-items: left;
      box-shadow: -17px 29px 5.63px rgba(0, 0, 0, 0.25);
      width: 15rem;
      height: 2rem;
      
    }
    .goal-input-box input {
      font-family: 'Road Rage', cursive;
      border: none;
      background: transparent;
      color: black;
      outline: none;
      width: 15rem;
      font-size: 2rem;
    }
    .fixed-text { 
      align-items: right; 
      color: rgba(0,0,0,0.74); 
      font-size: 2rem;
    }
    /* .goal-container button,
    .goal-display button,
    .goal-edit .edit-buttons button {
      margin-top: 10px;
    } */
    .goal-container button {
      background-color: #101010;
      color: #66FFED;
      border: none;
      padding: 5px 10px;
      cursor: pointer;
      border-top-left-radius: 0;
      border-top-right-radius: 0;
      border-bottom-left-radius: 10px;
      border-bottom-right-radius: 10px;
      box-shadow: -17px 29px 5.63px rgba(0, 0, 0, 0.25);
      font-family: 'Road Rage', cursive;
      font-size: 2rem;
      /* margin: -0.1rem 0.1rem; */
      width: 10rem;
    }
    .goal-display {
      transform: skew(1deg, -2deg);
      margin-left: 1rem;
      display: flex;
      align-items: left;
      justify-content: left;
      gap: 10px;
      background: transparent;
    }

    .goal-text {
      text-shadow: -4px 4px 5.63px rgba(0,0,0,0.25);
      font-size: 3.5rem;
    }

    .goal-display .edit-icon {
      background: none;
      border: none;
      cursor: pointer;
      font-size: 1.5rem;
      padding-bottom: 1rem;
      /* box-shadow: -17px 29px 5.63px rgba(0,0,0,0.25); */
    }
    .goal-edit { transform: skew(1deg, -2deg);  }
    .goal-edit .edit-buttons {
      display: flex;
      
      /* margin-top: 10px; */
    }
    .goal-edit .edit-buttons button:first-child {
      background-color: #101010;
      color: #66FFED;
      border: none;
      padding: 5px 10px;
      cursor: pointer;
      width: 7rem;
      border-top-left-radius: 0;
      border-top-right-radius: 0;
      border-bottom-left-radius: 14px;
      border-bottom-right-radius: 0;
    }
    .goal-edit .edit-buttons button:last-child {
      background-color: #66FFED;
      color: #101010;
      border: 1px solid #101010;
      padding: 5px 10px;
      cursor: pointer;
      width: 7rem;
      border-top-left-radius: 0;
      border-top-right-radius: 0;
      border-bottom-right-radius: 14px;
      border-bottom-left-radius: 0;
    }
  
    /* --- Reminder & Log Form (Bottom II.a.2) --- */
    .reminder-log {
      transform: skew(1deg, -2deg);
      background-color: #101010;
      border-radius: 35px;
      box-shadow: -28px 19px 4px rgba(0,0,0,0.25);
      padding: 15px;
      display: flex;
      flex-direction: column;
      gap: 20px;
      color: #66FFED;
    }
    .reminder {
       display: flex; 
       flex-direction: column; 
       gap: 10px; 
       padding: 1rem 1rem 1rem 1rem;
    }

    .reminder-top {
      display: flex;
      gap: 2rem;
    }
    .reminder-text { 
      font-size: 2.5rem; 
    }
    .timer-display { display: flex; 
      align-items: center; 
      margin-top: -0.5rem;
      gap: 10px;
      font-size: 3.5rem;
    }
    .icon-btn {
      background: none;
      border: none;
      cursor: pointer;
      color: #66FFED;
      font-size: 1.5rem;
    }
    .reminder-options {
      display: flex;
      gap: 3.5px;
      font-size: 1.25rem;
    }

    .reminder-option:hover {
      background-color: #66FFED;
      border: 1px solid #101010;
      color: #101010;
    }

    .reminder-options.disabled {
      opacity: 0.75;
      pointer-events: none;
    }
    .reminder-option {
      width: 77px;
      height: 64px;
      display: flex;
      align-items: center;
      justify-content: center;
      background-color: #101010;
      border: 1px solid #66FFED;
      cursor: pointer;
    }
    .reminder-option.selected {
      background-color: #66FFED;
      color: #101010;
    }
    .log-form { 
      display: flex; 
      flex-direction: column; 
      gap: 10px; 
      padding: 0rem 1rem 1rem 1rem ;
    }
    .log-top { 
      display: flex; 
      flex-direction: column;
      gap: 5px; 
      font-size: 2.5rem;
    }
    .log-top input {
      background-color: #101010;
      border: 1px solid #66FFED;
      border-radius: 0.5rem;
      color: #66FFED;
      padding: 5px;
      width: 15rem;
      height: 2rem;
      font-family: 'Road Rage', cursive;
      font-size: 1.5rem;
    }
    .log-bottom { 
      display: flex; 
      justify-content: space-between; 
      align-items: center; 
      font-size: 2.5rem;
    }
    .rating { 
      /* display: flex;  */
      align-items: left; 
      /* gap: 5px;  */
      /* flex-direction: column; */
    }

    .starsEtButton {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      gap: 6rem;
    }
    
    .stars { 
      display: flex; 
      margin-top: 1rem;
      gap: 2px; 
      font-size: 3rem;
    }
    .star { cursor: pointer; }

    .stargon:hover {
      fill: #66FFED;
    }

    .log-form button {
      background-color: #66FFED;
      color: #101010;
      font-size: 2rem;
      font-family: 'Road Rage', cursive;
      width: 5rem;
      height: 3.5rem;
      border: none;
      border-radius: 0.5rem;
      padding: 5px 10px;
      cursor: pointer;
    }
  
    /* --- Table (Right II.b) --- */
    .right { 
      width: 30rem; position: relative; 
    }

    .right table {
      
      width: 100%;
      background-color: #101010;
      color: #66FFED;
      border-collapse: collapse;
      box-shadow: 23px 20px 4px rgba(0,0,0,0.25);
      max-height: 10rem;
    }
    .table-wrapper {
  max-height: 35rem; /* adjust this value to match 7 rows' height */
  overflow-y:scroll;
  border-radius: 2rem;
  box-shadow: 23px 20px 4px rgba(0,0,0,0.25);

}

/* Make the table header sticky so it stays visible during scroll */
.right table thead {
  position: sticky;
  top: 0;
  background-color: #101010; /* same as table background */
  z-index: 2;
}

/* 
  .right table.emptyTable thead {
    border-bottom: none !important;
    border-collapse: collapse !important;
    box-shadow: 23px 20px 4px rgba(0,0,0,0.25)!important;
  } */

  .right th, .right td {
    border: 1px solid #66FFED;
    border-radius: 2rem;
    padding: 8px;
    text-align: center;
    font-size: 2rem;
    align-items: center;
  }

  .right th {
    font-weight: 100;
    font-size: 2rem;
    letter-spacing: 0.1rem;
  }

  

    .right th:last-child,
    .right td:last-child {
      border-right: none;
    }
    .logStars {
      padding-top: 0.25rem;
    }
    .logStar {
      /* padding-bottom: 1.2rem; */
      padding-right: 0.3rem;
    }

    .right tbody tr:last-child th,
    .right tbody tr:last-child td {
      border-bottom: none;
    }

    .right th { background-color: #101010; font-weight: bold; }
    .right thead { border-bottom: 1px solid #66FFED; border-radius: 2rem;}
    .table-icon {
      background: none;
      border: none;
      cursor: pointer;
      color: #66FFED;
      margin-left: 5px;
    }
    .edit-input {
      background-color: #101010;
      border: 1px solid #66FFED;
      border-radius: 4px;
      color: #66FFED;
      width: 50px;
      padding: 2px;
    }

    .save-btn {
      background-color: #66FFED;
      color: #101010;
      font-size: 1.5rem;
      font-family: 'Road Rage', cursive;
      cursor: pointer;
      border-radius: 0.5rem;
      border:none;
      width: 4rem;
      height: 2rem;
    }

    .table-reset {
      position: absolute;
      top: -20px;
      right: -20px;
    }
    .reset-icon {
      background-color: black;
      color: #66FFED;
      border: none;
      border-radius: 50%;
      width: 30px;
      height: 30px;
      cursor: pointer;
    }
  
    /* --- Prompt Overlay --- */
    .overlay {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      backdrop-filter: blur(5px);
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: 'Road Rage', cursive;
      font-size: 2rem;
      
    }
    .prompt {
      background-color: #101010;
      color: #66FFED;
      padding: 20px;
      border-radius: 10px;
      text-align: center;
      border: 0.5px solid #66FFED;

    }
    .prompt-buttons {
      margin-top: 20px;
      display: flex;
      justify-content: space-around;
      font-family: 'Road Rage', cursive;
      font-size: 1.5rem;
    }
    .prompt-buttons button {
      background-color: #101010;
      color: #66FFED;
      border: 1px solid #66FFED;
      padding: 5px 15px;
      cursor: pointer;
      border-radius: 5px;
      font-family: 'Road Rage', cursive;
      font-size: 1.5rem;
    }
    .prompt-buttons button:hover {
      background-color: #66FFED;
      color: #101010;
    }
  </style>
  