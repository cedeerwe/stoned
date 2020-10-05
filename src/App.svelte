<script lang="ts">
  import Icon from "svelte-awesome";
  import {
    faHeart,
    faAngry,
    faBell,
    faClock,
    faHandPeace,
    faMoon,
    faArrowAltCircleLeft,
    faArrowAltCircleRight,
  } from "@fortawesome/free-regular-svg-icons";
  import { faRedo } from "@fortawesome/free-solid-svg-icons";

  function sleep(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  enum CellType {
    Heart = "Heart",
    Angry = "Angry",
    Bell = "Bell",
    Clock = "Clock",
    // HandPeace = "HandPeace",
    // Moon = "Moon",
  }

  function getIcon(cellType: CellType) {
    switch (cellType) {
      case CellType.Heart:
        return faHeart;
      case CellType.Angry:
        return faAngry;
      case CellType.Bell:
        return faBell;
      case CellType.Clock:
        return faClock;
      // case CellType.Moon:
      //   return faMoon;
      // case CellType.HandPeace:
      //   return faHandPeace;
    }
  }

  type Cell = undefined | { type: CellType; player: boolean };
  let player = false;
  let symbol = CellType.Heart;
  const board_x = 4;
  const board_y = 4;
  const board: Cell[][] = [...Array(board_y)].map(() =>
    [...Array(board_x)].map(() => undefined)
  );
  let pastBoards = [copyBoard(board)];
  let finished = false;
  let round = 1;
  let results: Array<{ player: number; opponent: number }> = [];

  resetState();

  async function resetState() {
    for (let x = 0; x < board_x; x++) {
      for (let y = 0; y < board_y; y++) {
        board[x][y] = undefined;
      }
    }
    round = 1;
    symbol = CellType.Heart;
    player = false;
    finished = false;
    results = [];
    pastBoards = [copyBoard(board)];
    const { x, y, sym } = optimalStep(board);
    await sleep(1000);
    fillCell(x, y, sym);
    pastBoards.push(copyBoard(board));
    round += 1;
  }

  function playerMove(x: number, y: number, symbol: CellType) {
    return async () => {
      fillCell(x, y, symbol);
      // if player actually made a move
      if (!player) {
        round += 1;
        pastBoards.push(copyBoard(board));
        if (round > board_x * board_y) {
          finished = true;
        } else {
          await sleep(1000);
          const { x, y, sym } = optimalStep(board);
          fillCell(x, y, sym);
          pastBoards.push(copyBoard(board));
          round += 1;
          if (round > board_x * board_y) {
            finished = true;
          }
        }
      }
    };
  }

  function fillCell(x: number, y: number, symbol: CellType) {
    if (board[x][y] === undefined) {
      board[x][y] = { type: symbol, player };
      const result = {
        player: computeScore(board, true),
        opponent: computeScore(board, false),
      };
      results = [...results, result];
      player = !player;
    }
  }

  function setSymbol(cellType: CellType) {
    return () => {
      symbol = cellType;
    };
  }

  function computeScore(board: Cell[][], player: boolean): number {
    let score = 0;
    for (const [x, row] of board.entries()) {
      for (const [y, cell] of row.entries()) {
        if (cell !== undefined) {
          if (cell.type === CellType.Heart && player === cell.player) {
            score += 6;
          } else if (cell.type === CellType.Clock && player === cell.player) {
            if (
              x === 0 ||
              x === board.length - 1 ||
              y === 0 ||
              y === row.length - 1
            ) {
              score += 7;
            }
          } else if (cell.type === CellType.Angry && player !== cell.player) {
            for (const i of [-1, 0, 1]) {
              for (const j of [-1, 0, 1]) {
                const x2 = x + i;
                const y2 = y + j;
                if (x2 >= 0 && y2 >= 0 && x2 < board_x && y2 < board_y) {
                  const cell2 = board[x2][y2];
                  if (cell2 !== undefined && cell2.player === player) {
                    score -= 3;
                  }
                }
              }
            }
          } else if (cell.type === CellType.Bell && player === cell.player) {
            for (const i of [-1, 1]) {
              for (const j of [-1, 1]) {
                const x2 = x + i;
                const y2 = y + j;
                if (x2 >= 0 && y2 >= 0 && x2 < board_x && y2 < board_y) {
                  const cell2 = board[x2][y2];
                  if (
                    cell2 !== undefined &&
                    cell2.player === player &&
                    cell2.type === CellType.Bell
                  ) {
                    score += 5;
                  }
                }
              }
            }
          }
        }
      }
    }
    return score;
  }

  function availableSymbols(): CellType[] {
    return [...Object.values(CellType)];
  }

  function copyBoard(board: Cell[][]): Cell[][] {
    return [...board].map((r) => [...r]);
  }

  function optimalStep(
    sharedBoard: Cell[][]
  ): { x: number; y: number; sym: CellType } {
    const board = copyBoard(sharedBoard);
    let optimalScore = -Infinity;
    let optimalPlays: Array<{ x: number; y: number; sym: CellType }> = [];
    const symbols = availableSymbols();
    for (let x = 0; x < board.length; x++) {
      for (let y = 0; y < board[x].length; y++) {
        if (board[x][y] === undefined) {
          for (const sym of symbols) {
            // try to add the symbol to the given place
            board[x][y] = { type: sym, player: false };
            const playerScore = computeScore(board, true);
            const opponentScore = computeScore(board, false);
            const score = opponentScore - playerScore;
            // compare optimality of score
            if (score > optimalScore) {
              optimalScore = score;
              optimalPlays = [{ x, y, sym }];
            } else if (score === optimalScore) {
              optimalPlays.push({ x, y, sym });
            }
            // remove the symbol
            board[x][y] = undefined;
          }
        }
      }
    }
    if (optimalPlays.length === 0) {
      throw new Error("no optimal plays");
    }
    return optimalPlays[Math.floor(Math.random() * optimalPlays.length)];
  }
</script>

<style>
  button {
    width: 100%;
    height: 100%;
  }
  .cell-full-opponent {
    background-color: crimson;
  }
  .cell-full-opponent:active {
    background-color: darkred;
  }
  .cell-full-player {
    background-color: dodgerblue;
  }
  .cell-full-player:active {
    background-color: royalblue;
  }
  .cell-empty {
    background-color: white;
  }
  .table-holder {
    display: flex;
    justify-content: center;
    align-items: center;
    padding-top: 20px;
  }
  .symbol-holder {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 30px;
  }
  .results-holder {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 30px;
  }
  .score-holder {
    display: flex;
    justify-content: center;
    align-items: center;
    padding-top: 100px;
  }
  .symbol-button {
    height: 50px;
    width: 50px;
    background-color: skyblue;
  }
  .symbol-button-active {
    background-color: royalblue;
  }
  .history-button {
    height: 50px;
    width: 50px;
    background-color: lightgreen;
  }
  .history-button:active {
    background-color: green;
  }
  .player-board {
    border: 1px;
    border-collapse: collapse;
  }
  .board-row,
  .board-cell {
    padding: 0;
    height: 50px;
    width: 50px;
  }
  .score-opponent {
    color: crimson;
  }
  .score-player {
    color: dodgerblue;
  }
  .score {
    font-size: 30px;
  }
</style>

<div class="score-holder score">
  <span
    class="score-player">{results
      .slice(0, round - 1)
      .reduce((pv, cv) => pv + cv.player, 0)}</span>
  :
  <span
    class="score-opponent">{results
      .slice(0, round - 1)
      .reduce((pv, cv) => pv + cv.opponent, 0)}</span>
</div>

<div class="table-holder">
  <table class="player-board">
    {#each pastBoards[round - 1] as row, x}
      <tr class="board-row">
        {#each row as el, y}
          <td class="board-cell">
            <button
              on:click={playerMove(x, y, symbol)}
              class:cell-full-player={el !== undefined && el.player}
              class:cell-full-opponent={el !== undefined && !el.player}
              class:cell-empty={el === undefined}>
              {#if el !== undefined}
                <Icon data={getIcon(el.type)} scale="1.5" />
              {/if}</button>
          </td>
        {/each}
      </tr>
    {/each}
  </table>
</div>

<div class="symbol-holder">
  {#if finished}
    <button
      class="history-button"
      disabled={round === 1}
      on:click={() => (round = round - 1)}><Icon
        data={faArrowAltCircleLeft}
        scale="1.5" /></button>
    <button
      class="history-button"
      disabled={round === pastBoards.length}
      on:click={() => (round = round + 1)}><Icon
        data={faArrowAltCircleRight}
        scale="1.5" /></button>
    <button class="history-button" on:click={resetState}><Icon
        data={faRedo}
        scale="1.5" /></button>
  {:else}
    {#each Object.values(CellType) as sym}
      <button
        class="symbol-button"
        class:symbol-button-active={symbol === sym}
        on:click={setSymbol(sym)}><Icon
          data={getIcon(sym)}
          scale="1.5" /></button>
    {/each}
  {/if}
</div>

<div class="results-holder">
  <br />

  <table>
    <thead>
      <tr>
        <td>Round</td>
        <td>Player</td>
        <td>Opponent</td>
      </tr>
    </thead>
    <tbody>
      {#each results as result, i}
        <tr>
          <td>{i + 1}</td>
          <td>{result.player}</td>
          <td>{result.opponent}</td>
        </tr>
      {/each}
    </tbody>
  </table>
</div>
