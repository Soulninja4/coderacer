<script lang="ts">
  import { blur } from 'svelte/transition';
	import { onMount } from 'svelte';
  import { tweened } from 'svelte/motion'
  

  type Game = 'waiting for input' | 'in progress' | 'game over'
  type Word = string

  var game: Game = 'waiting for input'
  var seconds = 30
  var typedLetter = ''

  let words: Word[] = []

  var wordIndex = 0
  var letterIndex = 0
  var correctLetters = 0

  var wpm = tweened(0, { delay: 300, duration: 1000 } )
  var accuracy = tweened(0, { delay: 1300, duration: 1000 } )

  var wordsEl: HTMLDivElement
  var letterEl: HTMLSpanElement
  var inputEl: HTMLInputElement
  var cursorEl: HTMLDivElement

  var toggleReset = false

  function resetGame() {
    toggleReset = !toggleReset
    setGameState('waiting for input')
    getWords(100)
    focusInput()
    

    seconds = 30
    typedLetter = ''
    wordIndex = 0
    letterIndex = 0
    correctLetters = 0

    $wpm = 0
    $accuracy = 0
    

  }

  function getWPM() {
    const word = 5
    const minutes = 0.5
    return Math.floor(correctLetters / word / minutes)
  }

  function getAccuracy() {
    const totalLetters = getTotalLetters(words)
    return Math.floor(correctLetters / totalLetters * 100)
  }

  function getTotalLetters(words: Word[]) {
    return words.reduce((count, word) => count + word.length ,0)
  }

  function getResults() {
    $wpm = getWPM()
    $accuracy = getAccuracy()
    }

  function startGame() {
    setGameState('in progress')
    setGameTimer()
  }

  function setGameState(state: Game) {
    game = state
  }

  function setGameTimer() {
    function gameTimer() {
      if (seconds > 0) {
        seconds -= 1
      }

      if (game === 'waiting for input' || seconds === 0) {
        clearInterval(interval)
      }

      if (seconds === 0) {
        setGameState('game over')
        getResults()
      }
    }

    const interval = setInterval(gameTimer, 1000)
  }

  function updateGameState() {
    setLetter()
    checkLetter()
    nextLetter()
    updateLine()
    resetLetter()
    moveCursor()
  }

  function increaseScore() {
    correctLetters += 1
  }


  function setLetter() {
    const isWordCompleted = letterIndex > words[wordIndex].length - 1

    if (!isWordCompleted) {
      letterEl = wordsEl.children[wordIndex].children[letterIndex] as HTMLSpanElement
    }
    
  }

  function checkLetter() {
    const currentLetter = words[wordIndex][letterIndex]

    if (typedLetter === currentLetter) {
      letterEl.dataset.letter = 'correct'
      increaseScore()
    }

    if (typedLetter !== currentLetter) {
      letterEl.dataset.letter = 'incorrect'
    }
  }

  function nextLetter() {
    letterIndex+= 1
  }

  function updateLine() {
    const wordEl = wordsEl.children[wordIndex]
    const wordsY = wordsEl.getBoundingClientRect().y
    const wordY = wordEl.getBoundingClientRect().y

    if (wordY > wordsY) {
      wordEl.scrollIntoView({ block: 'center' })
    }
  }

  function resetLetter() {
    typedLetter = ''
  }

  function moveCursor() {
    const offset = 4
    cursorEl.style.top = `${letterEl.offsetTop + 4}px`
    cursorEl.style.left = `${letterEl.offsetLeft + letterEl.offsetWidth}px`
  }

  function nextWord() {
    const isNotFirstLetter = letterIndex !== 0
    const isOneLetterWord = words[wordIndex].length === 1

    if (isNotFirstLetter || isOneLetterWord) {
      wordIndex += 1
      letterIndex = 0
      // increaseScore()
      moveCursor()

    }
  }

  async function getWords(limit: number) {
    const response = await fetch(`/api/words?limit=${limit}`)
    words = await response.json()
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.code === 'Space') {
      e.preventDefault()

      if (game === 'in progress') {
        nextWord()
      }
    }

    if (game === 'waiting for input') {
      startGame()
    }
  }

  function focusInput() {
    inputEl.focus()
  }

  onMount(() => {
    getWords(100)
    focusInput()
  })

</script>

{#if game !== 'game over'}
<div class="game" data-game={game}>
  <input
    bind:this={inputEl}
    bind:value={typedLetter} 
    on:keydown={handleKeydown}
    on:input={updateGameState}
    type="text" 
    class="input"
  >

  <div class="time">{seconds}</div>

  {#key toggleReset}
  <div in:blur|local bind:this={wordsEl} class="words">
    {#each words as word}
      <span class="word">
        {#each word as letter}
          <span class="letter">{letter}</span>
        {/each}
      </span>
    {/each}
    <div bind:this={cursorEl} class="cursor" />
  </div>
  {/key}

  <div class="reset">
    <button on:click={resetGame} aria-label="reset">
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 24 24"
        width="24"
        height="24"
        stroke-width="1.5"
        stroke="currentColor"
        fill="none"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M15 15l6-6m0 0l-6-6m6 6H9a6 6 0 000 12h3"
        />
      </svg>
    </button>
  </div>
</div>
{/if}

{#if game === 'game over'}
  <div in:blur class="results">
    <div>
      <p class="title">wpm</p>
      <p class="score">{Math.trunc($wpm)}</p>
    </div>

    <div>
      <p class="title">accuracy</p>
      <p class="score">{Math.trunc($accuracy)}%</p>
    </div>
    <button on:click={resetGame} class="play">play again</button>
  </div>

{/if}

<style lang="scss">
  .game {
    position: relative;

    .time {
      margin: auto;
      text-align: center;
      position: relative;
      top: -48px;
      font-size: 1.5rem;
      color: var(--primary);
      opacity: 0;
      transition: all 0.3s ease;
    }

    .reset {
      width: 100%;
      display:grid;
      justify-content: center;
      margin-top: 2rem;
    }

    .input {
      position: absolute;
      opacity: 0;
    }

    &[data-game='in progress'] .time {
      opacity: 1;
    }
  }
  .words {
    --line-height: 1em;
    --lines: 3;

    margin: auto;
    width: 50%;
    max-height: calc(var(--line-height) * var(--lines) * 1.42);
    display: flex;
    flex-wrap: wrap;
    gap: 0.6em;
    position: relative;
    font-size: 1.5rem;
    line-height: var(--line-height);
    overflow: hidden;
    user-select: none;

    .letter {
      opacity: 0.4;
      transition: all 0.3s ease;

      &:global([data-letter='correct']) {
        opacity: 0.8;
      }

      &:global([data-letter='incorrect']) {
        color: var(--primary);
        opacity: 1;
      }
    }

    .cursor {
      position: absolute;
      height: 1.8rem;
      top: 0;
      border-right: 1px solid var(--primary);
      animation: cursor 1s infinite;
      transition: all 0.2s ease;

      @keyframes cursor {
      
        0%,
        to {
          opacity: 0;
        }
        50% {
          opacity: 1;
        }
      }
    }
  }

  .results {
    .title {
      font-size: 2rem;
      color: var(--fg-200);
    }

    .score {
      font-size: 4rem;
      color: var(--primary);
    }

    // .play {
    //   margin-top: 1rem;
    // }
  }
</style>
