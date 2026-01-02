
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <title>Die Chroniken der 7 Dörfer - Final Edition</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');
        body { font-family: 'VT323', monospace; background-color: #050505; color: #00ff00; overflow: hidden; }
        #game-container { max-width: 1000px; margin: 0 auto; height: 100vh; display: flex; flex-direction: column; padding: 20px; }
        #output { flex-grow: 1; overflow-y: auto; background: #000; border: 2px solid #222; padding: 20px; font-size: 1.3em; margin-bottom: 10px; border-radius: 5px; }
        #status-bar { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 15px; border: 2px solid #00ff00; padding: 15px; margin-bottom: 10px; background: #111; }
        .bar-bg { width: 100%; background: #333; height: 12px; border: 1px solid #555; margin-top: 5px; }
        #hp-bar { background: #ff4444; height: 100%; transition: 0.5s; }
        #mana-bar { background: #3b82f6; height: 100%; transition: 0.5s; }
        #xp-bar { background: #a855f7; height: 100%; transition: 0.5s; }
        #command-input { width: 100%; background: #000; border: 2px solid #00ff00; color: #00ff00; padding: 12px; font-family: 'VT323', monospace; font-size: 1.4em; outline: none; }
        .item-gold { color: #fbbf24; }
        .secret-item { color: #00ffff; text-shadow: 0 0 10px #00ffff; font-weight: bold; }
        .victory-screen { color: #fbbf24; text-align: center; border: 3px double #fbbf24; padding: 20px; margin-top: 20px; animation: glow 2s infinite; }
        @keyframes glow { 0% { box-shadow: 0 0 5px #fbbf24; } 50% { box-shadow: 0 0 20px #fbbf24; } 100% { box-shadow: 0 0 5px #fbbf24; } }
    </style>
</head>
<body>

<div id="game-container">
    <div id="status-bar">
        <div>
            <b>HELD:</b> <span id="stat-lvl">Lvl 1</span><br>
            <span class="text-yellow-400">GOLD: <span id="stat-taler">20</span></span>
        </div>
        <div>
            <b>HP:</b> <span id="hp-text">100/100</span>
            <div class="bar-bg"><div id="hp-bar" style="width: 100%"></div></div>
            <b>MANA:</b> <span id="mana-text">30/30</span>
            <div class="bar-bg"><div id="mana-bar" style="width: 100%"></div></div>
        </div>
        <div>
            <b>XP:</b> <span id="xp-text">0/100</span>
            <div class="bar-bg"><div id="xp-bar" style="width: 0%"></div></div>
            <small>ORT: <span id="current-loc">-</span></small>
        </div>
    </div>

    <div id="output"></div>

    <form id="command-form" class="mt-2">
        <input type="text" id="command-input" placeholder="Befehl eingeben (hilfe für Liste)..." autofocus autocomplete="off">
    </form>
</div>

<script>
    // --- SPIEL-KONFIGURATION ---
    let state = {
        lvl: 1, xp: 0, nextXp: 100, hp: 100, maxHp: 100, mana: 30, maxMana: 30, taler: 20, loc: 'lichtung',
        inventory: ['rostiges_schwert'], gameEnded: false,
        words: ['nord', 'sued', 'ost', 'west', 'hoch', 'runter', 'schlag', 'wucht', 'feuerball', 'suche', 'hilfe', 'trinken', 'reden', 'kaufen', 'inventar', 'essen', 'matsch', 'heiltrank', 'stahlschwert', 'stahlhelm', 'drachentoeter', 'bier', 'chilli', 'gulasch']
    };

    const prices = { 'heiltrank': 15, 'stahlschwert': 50, 'stahlhelm': 30, 'drachentoeter': 150, 'bier': 5, 'chilli': 5, 'gulasch': 10 };

    const map = {
        'lichtung': { n: 'Alte Lichtung', d: 'Ein friedlicher Startpunkt. Nord: Dorf Eichengrund.', e: { nord: 'eichengrund' } },
        'eichengrund': { n: 'Dorf Eichengrund', isV: true, d: 'Das erste Dorf. Ein Schmied und eine Taverne warten.', e: { sued: 'lichtung', nord: 'sumpf' }, shop: ['heiltrank', 'stahlschwert'], tavern: ['bier', 'chilli', 'gulasch'], tips: ['"Wusstest du? Chilli füllt dein Mana zur Hälfte auf!"'] },
        'sumpf': { n: 'Dunkler Sumpf', d: 'Modriger Schlamm überall. Ost: Sumpfende.', e: { sued: 'eichengrund', ost: 'sumpfend' }, enemies: [{ name: 'Sumpfratte', hp: 40, atk: 8, xp: 50 }] },
        'sumpfend': { n: 'Dorf Sumpfend', isV: true, d: 'Ein Fischerdorf am Sumpf.', e: { west: 'sumpf', nord: 'festung' }, shop: ['heiltrank', 'stahlhelm'], tavern: ['bier', 'chilli', 'gulasch'], tips: ['"Manche sagen, man könne im Matsch des Sumpfes geheime Dinge suchen..."'] },
        'festung': { n: 'Festung Ruine', d: 'Skelettkrieger bewachen diesen Ort.', e: { sued: 'sumpfend', nord: 'eisenstein' }, enemies: [{ name: 'Skelett', hp: 90, atk: 18, xp: 120 }] },
        'eisenstein': { n: 'Dorf Eisenstein', isV: true, d: 'Die Stadt der besten Schmiede.', e: { sued: 'festung', hoch: 'drachenfels' }, shop: ['stahlschwert', 'stahlhelm', 'drachentoeter'], tavern: ['bier', 'chilli', 'gulasch'], tips: ['"Nur mit dem Drachentöter-Schwert hast du eine echte Chance!"'] },
        'drachenfels': { n: 'Drachenfels', d: 'Kalter Wind weht. Nord: Die Drachenhöhle.', e: { runter: 'eisenstein', nord: 'hoehle' } },
        'hoehle': { n: 'Drachenhöhle', d: 'Überall verbrannte Knochen...', enemies: [{ name: 'FEUERDRACHE', hp: 600, atk: 35, xp: 1000, isBoss: true }], e: { sued: 'drachenfels' } }
    };

    // --- SYSTEM ---
    function print(txt) {
        const o = document.getElementById('output');
        o.innerHTML += `<div>${txt}</div>`;
        o.scrollTop = o.scrollHeight;
    }

    function update() {
        document.getElementById('stat-lvl').innerText = `Lvl ${state.lvl}`;
        document.getElementById('stat-taler').innerText = state.taler;
        document.getElementById('hp-text').innerText = `${state.hp}/${state.maxHp}`;
        document.getElementById('hp-bar').style.width = (state.hp/state.maxHp*100)+'%';
        document.getElementById('mana-text').innerText = `${state.mana}/${state.maxMana}`;
        document.getElementById('mana-bar').style.width = (state.mana/state.maxMana*100)+'%';
        document.getElementById('xp-text').innerText = `${state.xp}/${state.nextXp}`;
        document.getElementById('xp-bar').style.width = (state.xp/state.nextXp*100)+'%';
        document.getElementById('current-loc').innerText = map[state.loc].n;
    }

    // Tippfehler-Korrektur (Levenshtein)
    function fix(word) {
        if (!word || word.length < 3) return word;
        let best = word, min = 2;
        for(let w of state.words) {
            let d = (function(a,b){
                const m = Array.from({length: b.length+1},(_,i)=>[i]);
                for(let j=0;j<=a.length;j++)m[0][j]=j;
                for(let i=1;i<=b.length;i++)
                    for(let j=1;j<=a.length;j++)
                        m[i][j]=b[i-1]===a[j-1]?m[i-1][j-1]:Math.min(m[i-1][j-1]+1,m[i][j-1]+1,m[i-1][j]+1);
                return m[b.length][a.length];
            })(word, w);
            if(d < min) { min = d; best = w; }
        }
        return best;
    }

    function process(raw) {
        if(state.gameEnded) return;
        let parts = raw.toLowerCase().trim().split(/\s+/);
        let cmd = fix(parts[0]);
        let arg = fix(parts[1]);
        let room = map[state.loc];

        print(`<br><span style="color: #555">> ${raw}</span>`);
        if(cmd !== parts[0] && state.words.includes(cmd)) print(`<span class="text-red-400 italic">(Meintest du ${cmd}?)</span>`);

        if(['nord','sued','ost','west','hoch','runter'].includes(cmd)) {
            if(room.e[cmd]) { state.loc = room.e[cmd]; describe(); }
            else print("Weg versperrt.");
        }
        else if(cmd === 'hilfe') showHelp();
        else if(cmd === 'kaufen') handleShop(arg);
        else if(cmd === 'essen' || cmd === 'trinken') handleTavern(arg);
        else if(['bier','chilli','gulasch'].includes(cmd)) handleTavern(cmd);
        else if(cmd === 'reden' && room.isV) print(`<span class="text-blue-400">Bewohner: ${room.tips[0]}</span>`);
        else if(cmd === 'suche' && arg === 'matsch' && state.loc === 'sumpf') {
            if(!state.inventory.includes('laserpistole')) {
                state.inventory.push('laserpistole');
                print("<span class='secret-item'>UNGLAUBLICH! Du findest eine LASERPISTOLE im Schlamm!</span>");
            } else print("Hier ist nur Schlamm.");
        }
        else if(['schlag','wucht','feuerball'].includes(cmd)) battle(cmd);
        else if(cmd === 'inventar' || cmd === 'i') print("Beutel: " + state.inventory.join(', '));
        else if(cmd === 'benutze' && arg === 'heiltrank') {
            let i = state.inventory.indexOf('heiltrank');
            if(i > -1) { state.inventory.splice(i,1); state.hp = Math.min(state.maxHp, state.hp+50); print("Trank benutzt! +50 HP."); }
            else print("Kein Trank da.");
        }
        else print("Unbekannter Befehl.");
        update();
    }

    function handleShop(item) {
        let room = map[state.loc];
        if(!room.isV || !room.shop.includes(item)) return print("Das gibt es hier nicht.");
        if(state.taler >= prices[item]) {
            state.taler -= prices[item]; state.inventory.push(item);
            print(`<span class="item-gold">${item} gekauft!</span>`);
        } else print("Zu wenig Gold.");
    }

    function handleTavern(item) {
        let room = map[state.loc];
        if(!room.isV) return print("Keine Taverne hier.");
        if(!prices[item] || !['bier','chilli','gulasch'].includes(item)) return print("Was möchtest du? (bier, chilli, gulasch)");
        
        if(state.taler >= prices[item]) {
            state.taler -= prices[item];
            if(item === 'bier') { state.hp = state.maxHp; print("Bier getrunken! HP voll."); }
            else if(item === 'chilli') { state.mana = Math.min(state.maxMana, state.mana + Math.floor(state.maxMana/2)); print("Chilli gegessen! Mana regeneriert."); }
            else if(item === 'gulasch') { state.mana = state.maxMana; print("Gulasch gegessen! Mana voll."); }
        } else print("Zu wenig Gold.");
    }

    function battle(type) {
        let room = map[state.loc];
        if(!room.enemies || room.enemies.length === 0) return print("Kein Gegner da.");
        let enemy = room.enemies[0];
        let dmg = 0;

        if(state.inventory.includes('laserpistole')) {
            dmg = 9999;
            print("<span class='secret-item'>*PIU PIU* Laserstrahl vernichtet alles!</span>");
        } else {
            let bonus = state.inventory.includes('drachentoeter') ? 80 : (state.inventory.includes('stahlschwert') ? 15 : 0);
            if(type === 'schlag') dmg = 15 + bonus;
            else if(type === 'wucht') dmg = Math.random() > 0.4 ? 40 + (bonus*2) : 0;
            else if(type === 'feuerball') {
                if(state.mana >= 10) { state.mana -= 10; dmg = 80; }
                else return print("Mana leer!");
            }
        }

        enemy.hp -= dmg;
        print(`Angriff! ${dmg === 0 ? 'Verfehlt!' : dmg + ' Schaden.'}`);

        if(enemy.hp <= 0) {
            print(`${enemy.name} besiegt!`);
            if(enemy.isBoss) { showVictory(); }
            else {
                state.xp += enemy.xp; state.taler += 25; room.enemies.shift();
                if(state.xp >= state.nextXp) { state.lvl++; state.xp = 0; state.maxHp += 25; state.hp = state.maxHp; state.maxMana += 10; state.mana = state.maxMana; print("<b>LEVEL UP!</b>"); }
            }
        } else {
            let armor = state.inventory.includes('stahlhelm') ? 10 : 0;
            state.hp -= Math.max(1, enemy.atk - armor);
            if(state.hp <= 0) { alert("Besiegt... Spiel wird neu geladen."); location.reload(); }
        }
    }

    function showVictory() {
        state.gameEnded = true;
        print(`
            <div class="victory-screen">
                <h1 class="text-3xl font-bold">DER DRACHE IST BESIEGT!</h1>
                <p class="mt-4">Die Welt ist gerettet und du bist ein wahrer Held.</p>
                <hr class="my-4 border-yellow-600">
                <p class="text-xl">INFO: Das war erst der Anfang...</p>
                <p class="text-2xl font-bold animate-pulse">ES GEHT BALD MIT ANDEREN ABENTEUERN WEITER!</p>
            </div>
        `);
        document.getElementById('command-input').style.display = 'none';
    }

    function showHelp() {
        print(`<div style="color: #fbbf24">
<b>BEFEHLE:</b><br>
- <b>nord, sued, ost, west, hoch, runter</b>: Bewegen.<br>
- <b>schlag, wucht, feuerball</b>: Kämpfen.<br>
- <b>kaufen [Item]</b>: z.B. kaufen stahlschwert.<br>
- <b>trinken bier</b> (5 Gold): HP voll.<br>
- <b>essen chilli / gulasch</b> (5/10 Gold): Mana füllen.<br>
- <b>reden</b>: Bewohner befragen.<br>
- <b>inventar</b>: Sachen ansehen.<br>
- <b>suche matsch</b>: Geheimnisse finden.
</div>`);
    }

    function describe() {
        let r = map[state.loc];
        print(`<br><b class="text-blue-400">--- ${r.n} ---</b><br>${r.d}`);
        if(r.isV) {
            print(`<span class="item-gold">Marktplatz: ${r.shop.join(', ')}</span>`);
            print(`<span class="text-orange-400">Taverne: Bier (5), Chilli (5), Gulasch (10)</span>`);
        }
        if(r.enemies && r.enemies.length > 0) print(`<span class="text-red-500">FEIND: ${r.enemies[0].name} (HP: ${r.enemies[0].hp})</span>`);
    }

    document.getElementById('command-form').onsubmit = (e) => {
        e.preventDefault();
        let i = document.getElementById('command-input');
        if(i.value) process(i.value);
        i.value = '';
    };

    update(); describe();
    print("<b>DAS ABENTEUER BEGINNT!</b> Tippe 'hilfe' für Befehle.");
</script>
</body>
</html>
