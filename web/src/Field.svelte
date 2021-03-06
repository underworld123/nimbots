<script lang="ts">
  import fetch from "cross-fetch";
  import type { OpenWhisk } from "./openwhisk";
  import { URL_LOGIN, VERSION } from "./const";
  import { BattleWeb } from "./battleweb";
  import { AssetsLoader } from "./util";
  import { onMount, afterUpdate, onDestroy } from "svelte";
  import { inspector, source, submitting, board } from "./store";
  import { log } from "./robot";
  import { rumblePublic } from "./rumble";
  import Submit from "./Submit.svelte";
  import Share from "./Share.svelte";
  import type { Battle } from "./battle";

  export let base: string;
  export let ow: OpenWhisk;

  let battle: BattleWeb;
  let msg =
    ow === undefined ? "in a galaxy far far away..." : "Choose opponents";
  let status = "Select Opponents";

  let ready = false;
  let speed = BattleWeb.speed;
  let debug = false;
  let extra = "";

  let myBot: string;
  let enemyBot: string;

  let fighting = false;
  let editing = false;

  let robotName = "";
  let robotType = "js";

  let myBots: string[] = [];

  let enemyBots: { name: string; url: string }[] = [];
  let cyanBots = enemyBots;
  let redBots = enemyBots;

  let robotMap = {
    js: "/src/JsBot.js",
    go: "/src/GoBot.go",
    py: "/src/PyBot.py",
  };
  let regex = /^\w{1,60}$/g;

  async function create(): Promise<boolean> {
    if (!robotName.match(regex)) {
      alert("Invalid Robot Name");
      return false;
    }
    let bot: string;
    return fetch(robotMap[robotType])
      .then((data) => {
        if (data.ok) return data.text();
        throw data.statusText;
      })
      .then((code) => {
        bot = robotName + "." + robotType;
        return ow.save(bot, code, false);
      })
      .then(async (result) => {
        console.log(result);
        if ("error" in result) throw result["error"];
        source.set(bot);
        return true;
      })
      .catch((err) => {
        alert(err);
        return false;
      });
  }

  async function updateBots() {
    enemyBots = await rumblePublic();
    cyanBots = Object.assign([], enemyBots);
    cyanBots.sort(() => 0.5 - Math.random());
    redBots = Object.assign([], enemyBots);
    redBots.sort(() => 0.5 - Math.random());
    if (ow !== undefined) {
      myBots = await ow.list();
      if (myBots.length > 0) myBot = myBots[0];
    } else {
      myBot = cyanBots[0].url;
    }
    enemyBot = redBots[0].url;
  }

  let unsubscribeSource = source.subscribe((value) => {
    editing = value != "";
    updateBots();
  });

  
  function finish(winner: number) {
    msg = "Game over";
    if (winner == -2) {
      image = "ready";
      extra = "";
    } else if (winner == -1) {
      image = "draw";
      extra = "";
    } else if (winner == 0) {
      image = "won";
      extra = "Great Achievement! Share it with your friends!";
    } else {
      image = "lose";
      extra = "";
    }
    status = "Select Opponents";
    ready = false;
    fighting = false;
    battle.stop();
    inspector.set([
      { n: 0, req: "", res: "", state: "" },
      { n: 0, req: "", res: "", state: "" },
    ]);
  }

  function trace() {
    status = "Tracing...";
    fighting = false;
    msg = battle.trace();
  }

  function suspended(msg: string, state0: string, state1: string) {
    status = msg;
    fighting = false;
    inspector.update((info) => {
      info[0].state = state0;
      info[1].state = state1;
      return info;
    });
  }

  function edit() {
    console.log(myBot);
    source.set(myBot);
    battle.stop();
    editing = true;
  }

  let image = ow === undefined ? "splash" : "ready";
  let Images = new AssetsLoader({
    splash: "img/splash.png",
    ready: "img/ready.png",
    lose: "img/lose.png",
    won: "img/won.png",
    draw: "img/draw.png",
  });

  function splash() {
    //console.log("splash")
    let canvas = document.getElementById("arena") as HTMLCanvasElement;
    let ctx = canvas.getContext("2d");
    ctx.clearRect(0, 0, 500, 500);
    ctx.drawImage(Images.get(image), 0, 0);
  }

  afterUpdate(() => {
    if (!(editing || ready)) splash();
  });

  function selected() {
    let champ =
      myBots.length == 0
        ? myBot
        : ow.namespace + "/default/" + myBot.split(".")[0];

    let urls = [base + champ, base + enemyBot];
  
    console.log(urls);
    let canvas = document.getElementById("arena") as HTMLCanvasElement;

    let startAngles = [
      [Math.random() * 360, Math.random() * 360],
      [Math.random() * 360, Math.random() * 360],
    ];

    battle.webinit(canvas.getContext("2d"), urls, startAngles);
    ready = true;
    msg = "Starfighters in position!";
    status = "Ready to fight.";
    battle.draw();
  }

  function toggle() {
    fighting = !fighting;
    if (fighting) {
      status = "Fighting!";
      msg = battle.start();
    } else {
      status = "Suspended...";
      battle.stop();
    }
  }

  onMount(() => {
    let canvas = document.getElementById("arena") as HTMLCanvasElement;
    battle = new BattleWeb(
      parseInt(canvas.getAttribute("width")),
      parseInt(canvas.getAttribute("height")),
      finish,
      suspended
    );
    updateBots();
    Images.loadAll(() => splash());
  });
  onDestroy(unsubscribeSource);
</script>

<style>
  #arena {
    border: 1px solid grey;
    float: left;
  }

  #cyan {
    color: rgb(66, 168, 205);
  }
  #red {
    color: rgb(211, 19, 19);
  }
</style>

<main class="wrapper">
  <section class="container">
    <h1>{msg}</h1>
    <div class="row"><canvas id="arena" width="500" height="500" /></div>
    {#if $submitting != ''}
      <Submit {ow} />
    {:else if !ready}
      <div class="row">
        <h3>
          <a
            href="-"
            on:click={(event) => {
              console.log('click');
              board.set({ show: true, round: '' });
              event.preventDefault();
            }}>Leaderboard</a>
        </h3>
      </div>
      <div class="row">
        <div class="column column-left column-offset">
          <label for="enemy">Red Fighter (Enemy)</label>
          <select bind:value={enemyBot} id="enemy">
            {#each redBots as enemy}
              <option value={enemy.url}>{enemy.name}</option>
            {/each}
          </select>
        </div>
        <div class="column column-right">
          <label for="mybot">Cyan Fighter (You)</label>
          <select bind:value={myBot} id="enemy">
            {#if myBots.length == 0}
              {#each cyanBots as enemy}
                <option value={enemy.url}>{enemy.name}</option>
              {/each}
            {:else}
              {#each myBots as bot}
                <option value={bot}>{bot.split('.')[0]}</option>
              {/each}
            {/if}
          </select>
        </div>
      </div>
      <div class="row">
        <div class="column column-left column-offset">
          <button id="done" on:click={selected}>Start the Battle</button>
        </div>
        <div class="column column-right">
          {#if ow === undefined}
            <button
              id="login"
              on:click={() => {
                location.href = URL_LOGIN;
              }}>Login to Nimbella</button>
          {:else}
            <div class="column column-right">
              <button
                id="edit"
                on:click={edit}
                disabled={myBots.length == 0}>Edit my Fighter</button>
            </div>
          {/if}
        </div>
      </div>
      {#if ow === undefined}
        <div class="row">
          <div class="column column-center column-offset">
            <h4>
              Welcome to
              <b><a href="https://faaswars.nimbella.com">FAAS Wars</a></b>
              v{VERSION}. Please sign up and login to Nimbella to create and
              edit your starfighters.<br />
            </h4>
            <Share
              message="FAAS WARS: Code your fighter, learn serverless and win cash prize."
              url="https://faaswars.nimbella.com" />
          </div>
        </div>
      {:else}
        <div class="row">
          <div class="column column-left column-offset">
            <button id="create" on:click={create}>Create New Fighter</button>
          </div>
          <div class="column column-right">
            <input
              type="text"
              bind:value={robotName}
              placeholder="robot name"
              id="botname" />
          </div>
        </div>
        <div class="row">
          <div class="column column-left column-offset">
            <button
              id="submit"
              disabled={myBots.length == 0}
              on:click={() => {
                submitting.set(myBot);
              }}>Submit to FAAS WARS</button>
          </div>
          <div class="column column-right">
            <select bind:value={robotType}>
              <option value="js">JavaScript</option>
              <option value="py">Python</option>
              <option value="go">Golang</option>
            </select>
          </div>
        </div>
        <h4>{extra}</h4>
        <Share />
      {/if}
    {:else}
      <div class="row">
        <h3>{status}</h3>
      </div>
      <div class="row">
        <h4>
          You:
          <span id="cyan">{battle.robotName(0)}</span><br>
          Enemy:
          <span id="red">{battle.robotName(1)}</span>
        </h4>
      </div>
      <div class="row">
        <div class="column column-left column-offset">
          <br />
          <button id="fight" on:click={toggle}>
            {#if fighting}Suspend{:else}Fight!{/if}
          </button>
          <br />
          <button
            on:click={() => {
              ready = false;
              fighting = false;
              battle.terminate();
            }}>Stop</button><br />
          <button
            id="edit"
            on:click={edit}
            disabled={myBots.length == 0}>Edit</button>
        </div>
        <div class="column column-right">
          <br />
          <label>
            <input type="checkbox" bind:checked={debug} />
            Debug<br />
            <a
              href="https://apigcp.nimbella.io/wb/?command=activation+list"
              target="workbench">Logs</a>
          </label><br />
        </div>
      </div>
      {#if debug}
        <div class="row">
          <div class="column column-left column-offset">
            <button id="step" on:click={trace}>Trace</button>
          </div>
          <div class="column column-right">
            Trace:&nbsp;
            <label>
              <input type="checkbox" bind:checked={log.eventOn} />
              Events&nbsp;
            </label>
            <label>
              <input type="checkbox" bind:checked={log.requestOn} />
              Requests&nbsp;
            </label>
            <label>
              <input type="checkbox" bind:checked={log.actionOn} />
              Actions&nbsp;
            </label>
            (open console)
          </div>
        </div>
        <div class="row">
          <div class="column column-50 column-offset">
            <b>[Me] {$inspector[0].state}</b><br />
            Request/<b>Response</b>
            #{$inspector[0].n}
            <pre>{$inspector[0].req}<br /><b>{$inspector[0].res}</b>
              </pre>
            <b>[Emeny] {$inspector[1].state}</b><br />
            Request/<b>Response</b>
            #{$inspector[1].n}
            <pre>{$inspector[1].req}<br /><b>{$inspector[1].res}</b>
              </pre>
          </div>
        </div>
      {/if}
    {/if}
  </section>
</main>
