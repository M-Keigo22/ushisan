---
title: Welcome to pokemonshiritori
---
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>ポケモンシリトリ</title>
  <style>
    body {
      margin: 0;
      font-family: 'Arial', sans-serif;
      background: linear-gradient(135deg, #f9f9f9, #b2d4f8);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
    }
    h1 {
      font-size: 3rem;
      margin-bottom: 10px;
    }
    select, button, input {
      margin: 5px;
      padding: 10px;
      font-size: 1rem;
    }
    #gameArea {
      display: none;
      flex-direction: column;
      align-items: center;
      margin-top: 20px;
    }
    #log {
      background: white;
      border-radius: 10px;
      padding: 10px;
      width: 320px;
      height: 220px;
      overflow-y: auto;
      white-space: pre-line;
      box-shadow: 0 0 5px #aaa;
      font-weight: bold;
      font-size: 1.1rem;
    }
    #timer {
      font-size: 2rem;
      color: #d33;
      margin-top: 10px;
      font-weight: bold;
    }
    #loseMessage {
      font-size: 3rem;
      color: red;
      font-weight: bold;
      margin-top: 20px;
      display: none;
    }
  </style>
</head>
<body>
  <h1>ポケモンシリトリ</h1>
  <select id="mode">
    <option value="easy">イージー（制限なし・「ン」で終わる単語優先）</option>
    <option value="normal" selected>ノーマル（制限60秒）</option>
    <option value="hard">ハード（制限30秒）</option>
    <option value="ultimate">アルティメイト（制限20秒）</option>
    <option value="nightmare">ナイトメア（制限10秒）</option>
  </select>
  <button onclick="startGame()">▶ ゲームをはじめる</button>

  <div id="gameArea">
    <input id="userInput" placeholder="単語を入力" autocomplete="off" />
    <button onclick="play()">送信</button>
    <div id="log"></div>
    <div id="timer"></div>
    <div id="loseMessage">あなたの負け！時間切れ！</div>
  </div>

  <script src="https://unpkg.com/wanakana"></script>
  <script>
    const pokemonUrl = "https://gist.githubusercontent.com/PonDad/93922f63c3143489e30c3716d3d176d2/raw/pokemon.json";

    let wordList = [];
    let usedWords = [];
    let lastChar = '';
    let timerId = null;
    let timeLeft = 0;

    function toKatakana(str) {
      return wanakana.toKatakana(str);
    }

    function normalizeChar(char) {
      const smallToLarge = {'ァ': 'ア','ィ': 'イ','ゥ': 'ウ','ェ': 'エ','ォ': 'オ','ャ': 'ヤ','ュ': 'ユ','ョ': 'ヨ','ッ': 'ツ'};
      return smallToLarge[char] || char;
    }

    function getLastChar(word) {
      let char = word[word.length - 1];
      if (char === 'ー') {
        char = word[word.length - 2] || '';
      }
      return normalizeChar(char);
    }

    async function loadPokemonWords() {
      const res = await fetch(pokemonUrl);
      const data = await res.json();
      return data.map(p => toKatakana(p.ja));
    }

    function log(msg) {
      const logBox = document.getElementById('log');
      logBox.textContent += msg + '\n';
      logBox.scrollTop = logBox.scrollHeight;
    }

    function resetTimer() {
      clearInterval(timerId);
      document.getElementById('timer').textContent = '';
      document.getElementById('loseMessage').style.display = 'none';
    }

    function startTimer(seconds) {
      if (!seconds) return;
      timeLeft = seconds;
      document.getElementById('timer').textContent = `残り時間: ${timeLeft}秒`;
      timerId = setInterval(() => {
        timeLeft--;
        document.getElementById('timer').textContent = `残り時間: ${timeLeft}秒`;
        if (timeLeft <= 0) {
          clearInterval(timerId);
          document.getElementById('loseMessage').style.display = 'block';
          document.getElementById('userInput').disabled = true;
        }
      }, 1000);
    }

    async function startGame() {
      wordList = await loadPokemonWords();
      usedWords = [];
      lastChar = '';
      resetTimer();

      document.querySelector('select').style.display = 'none';
      document.querySelector('button').style.display = 'none';
      document.getElementById('gameArea').style.display = 'flex';
      document.getElementById('log').textContent = '';
      document.getElementById('userInput').disabled = false;
      document.getElementById('userInput').focus();

      const mode = document.getElementById('mode').value;
      if (mode === 'normal') startTimer(60);
      else if (mode === 'hard') startTimer(30);
      else if (mode === 'ultimate') startTimer(20);
      else if (mode === 'nightmare') startTimer(10);
    }

    function chooseAIWord(mode, lastChar, usedWords, wordList) {
      let candidates = wordList.filter(w => normalizeChar(w[0]) === lastChar && !usedWords.includes(w));

      if (candidates.length === 0) return null;

      if (mode === 'easy') {
        const nEndWords = candidates.filter(w => w.endsWith('ン'));
        if (nEndWords.length) return nEndWords[0];
        return candidates[0];
      }

      if (mode === 'normal') return candidates[0];

      let bestWord = null;
      let minNextCount = Infinity;
      for (const w of candidates) {
        if (w.endsWith('ン')) continue;
        const nextChar = getLastChar(w);
        const nextCandidates = wordList.filter(x => normalizeChar(x[0]) === nextChar && !usedWords.includes(x));
        if (nextCandidates.length < minNextCount) {
          minNextCount = nextCandidates.length;
          bestWord = w;
        }
      }
      return bestWord || candidates[0];
    }

    function play() {
      if (document.getElementById('loseMessage').style.display === 'block') return;

      const input = document.getElementById('userInput');
      const rawWord = input.value.trim();
      const userWord = toKatakana(rawWord); // 修正：余計な toHiragana を除去
      input.value = '';

      if (!userWord) return;

      const mode = document.getElementById('mode').value;

      if (!wordList.includes(userWord)) {
        log(`あなた：${userWord}\n→ ポケモンの名前ではありません`);
        return;
      }

      if (userWord.endsWith('ン')) {
        log(`あなた：${userWord}\n→「ン」で終わり！あなたの負け！`);
        clearInterval(timerId);
        document.getElementById('userInput').disabled = true;
        return;
      }

      if (lastChar && normalizeChar(userWord[0]) !== lastChar) {
        log(`あなた：${userWord}\n→ 「${lastChar}」で始まる単語を入力してください`);
        return;
      }

      if (usedWords.includes(userWord)) {
        log(`あなた：${userWord}\n→ その単語はすでに使われました`);
        return;
      }

      usedWords.push(userWord);
      lastChar = getLastChar(userWord);

      const aiWord = chooseAIWord(mode, lastChar, usedWords, wordList);
      if (!aiWord) {
        log(`あなた：${userWord}\nAI：もう思いつかない…あなたの勝ち！`);
        clearInterval(timerId);
        document.getElementById('userInput').disabled = true;
        return;
      }

      usedWords.push(aiWord);
      lastChar = getLastChar(aiWord);

      if (aiWord.endsWith('ン')) {
        log(`あなた：${userWord}\nAI：${aiWord}\n→「ン」で終わり！AIの負け！`);
        clearInterval(timerId);
        document.getElementById('userInput').disabled = true;
        return;
      }

      log(`あなた：${userWord}\nAI：${aiWord}`);

      resetTimer();
      if (mode === 'normal') startTimer(60);
      else if (mode === 'hard') startTimer(30);
      else if (mode === 'ultimate') startTimer(20);
      else if (mode === 'nightmare') startTimer(10);
    }

    document.getElementById('userInput').addEventListener('keydown', e => {
      if (e.key === 'Enter') play();
    });
  </script>
</body>
</html>

