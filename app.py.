import os, re, json, random, string, threading, urllib.request
from flask import Flask, request, jsonify, render_template_string

app = Flask(__name__)
PORT = int(os.environ.get("PORT", 10000))
# ВЕСУМ: список словоформ, правильних з погляду правопису.
# Якщо URL зміниться, можна покласти words_spell.txt поруч із app.py.
DICT_URLS = [
    "https://raw.githubusercontent.com/brown-uk/dict_uk/master/out/words_spell.txt",
]
ALPHABET = "АБВГҐДЕЄЖЗИІЇЙКЛМНОПРСТУФХЦЧШЩЬЮЯ"
POINTS = dict(zip(ALPHABET, [1,3,2,3,5,2,1,3,5,4,1,1,3,4,2,2,2,1,1,2,1,1,1,2,8,4,5,5,5,8,3,5,3]))
COUNTS = dict(zip(ALPHABET, [8,2,4,2,1,3,7,1,1,2,5,6,1,1,4,3,3,6,8,3,4,4,5,4,1,1,1,1,1,1,1,1,3]))
dictionary = set()
dict_status = "Завантаження словника…"
rooms = {}
lock = threading.RLock()

def norm(s):
    return s.upper().replace("’", "'").replace("'", "").strip()

def load_dictionary():
    global dictionary, dict_status
    local = os.path.join(os.path.dirname(__file__), "words_spell.txt")
    try:
        if os.path.exists(local):
            with open(local, encoding="utf-8") as f:
                source = f.read()
        else:
            source = None
            for url in DICT_URLS:
                try:
                    req = urllib.request.Request(url, headers={"User-Agent":"UkrErudit/1.0"})
                    with urllib.request.urlopen(req, timeout=60) as r:
                        source = r.read().decode("utf-8-sig")
                    break
                except Exception:
                    pass
        if source is None:
            raise RuntimeError("Не вдалося завантажити ВЕСУМ. Можна додати words_spell.txt поруч із app.py.")
        words = set()
        for line in source.splitlines():
            w = norm(line.split()[0]) if line.split() else ""
            if 2 <= len(w) <= 40 and all(ch in ALPHABET for ch in w):
                words.add(w)
        if len(words) < 1000:
            raise RuntimeError("Отриманий файл не схожий на повний словник.")
        dictionary = words
        dict_status = f"Словник готовий: {len(dictionary):,} словоформ"
    except Exception as e:
        dict_status = f"Словник не завантажено: {e}"
        # Невеликий резервний словник, щоб можна було перевірити саму гру.
        dictionary = {norm(x) for x in """АБИ АЖ АЗ БІК БІЛЬ БУКВА ВОДА ДІМ ДІД ДЕНЬ ДУБ ЖАР ЗАЄЦЬ ЗІРКА КІТ КІШКА КНИГА КОЛО КІНЬ ЛІС ЛІТО ЛЮДИ МАК МАМА МИР МІСТО МОВА МОРЕ НІЧ НІС НОГА ОКО ОРЕЛ ПАРА ПЕС ПІСНЯ РІКА РІК РУКА САД СИЛА СИН СІЛЬ СЛОВО СОН СТІЛ СТІНА СУМ ТАТО ТИРЕ ТІНЬ ХАТА ХЛІБ ХМАРА ЦУКОР ЧАС ШУМ ЯБЛУКО УКРАЇНА МАШИНА СОБАКА КОМПЮТЕР""".split()}

threading.Thread(target=load_dictionary, daemon=True).start()

def new_bag():
    bag=[]
    for letter,n in COUNTS.items():
        bag += [letter]*n
    random.shuffle(bag)
    return bag

def draw_tiles(room, player):
    rack=room["players"][player]["rack"]
    while len(rack)<7 and room["bag"]:
        rack.append(room["bag"].pop())

def snapshot(room, pid):
    p=room["players"][pid]
    return {
        "code":room["code"], "players":[{"name":x["name"],"score":x["score"],"joined":x["joined"]} for x in room["players"]],
        "you":pid, "turn":room["turn"], "board":room["board"], "rack":p["rack"],
        "bag_count":len(room["bag"]), "status":room["status"], "last_message":room["last_message"],
        "dict_status":dict_status, "started":room["started"]
    }

def get_line(board, r, c, dr, dc):
    rr,cc=r,c
    while 0<=rr-dr<15 and 0<=cc-dc<15 and board[rr-dr][cc-dc]:
        rr-=dr; cc-=dc
    word=""; cells=[]
    while 0<=rr<15 and 0<=cc<15 and board[rr][cc]:
        word += board[rr][cc]["l"]; cells.append((rr,cc))
        rr+=dr; cc+=dc
    return word,cells

HTML = r"""<!doctype html>
<html lang="uk"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Український Ерудит</title>
<style>
*{box-sizing:border-box}body{margin:0;background:#17212b;color:#f5f7fa;font-family:Arial,sans-serif}
.wrap{max-width:1100px;margin:auto;padding:18px}.top{text-align:center}.top h1{margin:8px;color:#f3c66b}
.panel{background:#243342;border-radius:14px;padding:14px;margin:12px 0}
button,input{font-size:16px;border:0;border-radius:9px;padding:11px;margin:4px}
button{background:#e8b85e;color:#17212b;font-weight:bold;cursor:pointer}button:disabled{opacity:.4;cursor:not-allowed}
input{max-width:100%;background:#fff}.muted{color:#b8c5d1;font-size:13px}
#game{display:none}.layout{display:grid;grid-template-columns:minmax(0,1fr) 230px;gap:14px}
.board{display:grid;grid-template-columns:repeat(15,minmax(20px,1fr));gap:2px;background:#536474;padding:4px;border-radius:8px}
.cell{aspect-ratio:1;background:#e8d9b9;color:#17212b;border-radius:3px;display:flex;align-items:center;justify-content:center;font-weight:bold;position:relative;cursor:pointer;min-width:0}
.cell.center{background:#e6a34a}.cell.tile{background:#fff0cc;box-shadow:inset 0 0 0 2px #9b7848}.cell small{position:absolute;right:2px;bottom:1px;font-size:8px}
.rack{display:flex;justify-content:center;gap:6px;flex-wrap:wrap;margin:12px 0}.tile{width:42px;height:46px;background:#f4c875;color:#222;border-radius:6px;display:flex;align-items:center;justify-content:center;font-weight:bold;font-size:21px;cursor:pointer;border:2px solid transparent}.tile.sel{border-color:#65dfa0;transform:translateY(-4px)}
.side h3{margin:4px 0 10px}.player{padding:10px;background:#33495b;border-radius:8px;margin:6px 0}.active{outline:2px solid #65dfa0}
#msg{min-height:22px;color:#9fe0b8}.actions{display:flex;flex-wrap:wrap;justify-content:center}
@media(max-width:760px){.layout{grid-template-columns:1fr}.side{display:grid;grid-template-columns:1fr 1fr;gap:8px}.board{gap:1px;padding:2px}.wrap{padding:8px}.cell{font-size:12px}.cell small{font-size:6px}}
</style></head><body><div class="wrap">
<div class="top"><h1>🇺🇦 УКРАЇНСЬКИЙ ЕРУДИТ</h1><div class="muted">Грай удвох онлайн — створи кімнату та надішли посилання другу</div></div>
<div id="lobby" class="panel">
<h3>Створити нову гру</h3><input id="name" placeholder="Твоє ім’я" maxlength="18"><button onclick="createRoom()">Створити кімнату</button>
<hr style="border-color:#465767"><h3>Приєднатися до гри</h3><input id="code" placeholder="Код кімнати" maxlength="6"><button onclick="joinRoom()">Приєднатися</button>
<p class="muted" id="lobbymsg">Завантаження словника…</p></div>
<div id="game" class="panel"><div style="display:flex;justify-content:space-between;flex-wrap:wrap;gap:8px"><b id="roomtitle"></b><button onclick="shareLink()">Копіювати посилання для друга</button></div>
<p id="msg"></p><div class="layout"><div><div id="board" class="board"></div><div class="rack" id="rack"></div>
<div class="actions"><button onclick="submitMove()">Підтвердити хід</button><button onclick="undoMove()">Повернути літери</button><button onclick="passTurn()">Пропустити хід</button></div>
<div class="muted" style="text-align:center">Вибери літеру внизу, потім клітинку на полі. Перше слово має проходити через центр.</div></div>
<div class="side"><div><h3>Гравці</h3><div id="players"></div></div><div><h3>Гра</h3><div class="player">Літер у банку: <b id="bag"></b></div><div class="player" id="dict"></div><div class="muted">Словник: ВЕСУМ. Для приватного некомерційного використання з атрибуцією.</div></div></div></div></div>
</div>
<script>
let room=null, me=null, state=null, selected=null, pending=[], timer=null;
const $=id=>document.getElementById(id);
function say(t){$('msg').textContent=t}
async function api(path,data={}){let r=await fetch(path,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(data)});let j=await r.json();if(!r.ok)throw Error(j.error||'Помилка');return j}
async function createRoom(){try{let j=await api('/api/create',{name:$('name').value||'Гравець 1'});room=j.code;me=j.pid;enter()}catch(e){$('lobbymsg').textContent=e.message}}
async function joinRoom(){try{let j=await api('/api/join',{code:$('code').value.toUpperCase(),name:$('name').value||'Гравець 2'});room=j.code;me=j.pid;enter()}catch(e){$('lobbymsg').textContent=e.message}}
function enter(){$('lobby').style.display='none';$('game').style.display='block';history.replaceState({},'',`?room=${room}`);$('roomtitle').textContent='Кімната '+room;refresh();if(timer)clearInterval(timer);timer=setInterval(refresh,1300)}
async function refresh(){if(!room)return;try{state=await (await fetch(`/api/state/${room}/${me}`)).json();if(state.error){say(state.error);return}render()}catch(e){}}
function render(){
 $('dict').textContent=state.dict_status;$('bag').textContent=state.bag_count;
 $('players').innerHTML=state.players.map((p,i)=>`<div class="player ${i===state.turn?'active':''}">${esc(p.name)}<br>Очки: <b>${p.score}</b> ${p.joined?'':'(очікує)'}</div>`).join('');
 let b=$('board');b.innerHTML='';
 for(let r=0;r<15;r++)for(let c=0;c<15;c++){let t=state.board[r][c],d=document.createElement('div');d.className='cell'+(r===7&&c===7?' center':'')+(t?' tile':'');d.textContent=t?t.l:((r===7&&c===7)?'★':'');if(t){let sm=document.createElement('small');sm.textContent=t.p;d.appendChild(sm)}d.onclick=()=>place(r,c);b.appendChild(d)}
 let rack=$('rack');rack.innerHTML='';state.rack.forEach((l,i)=>{let d=document.createElement('div');d.className='tile'+(selected===i?' sel':'');d.textContent=l;d.onclick=()=>{selected=i;render()};rack.appendChild(d)});
 let myTurn=state.turn===me;
 say(state.last_message+(myTurn?' — твій хід':' — хід суперника'));
 document.querySelectorAll('.actions button').forEach(x=>x.disabled=!myTurn);
}
function esc(s){return String(s).replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]))}
async function place(r,c){if(state.turn!==me||selected===null)return;try{let j=await api('/api/place',{room, pid:me,index:selected,r,c});selected=null;await refresh();if(j.error)say(j.error)}catch(e){say(e.message)}}
async function submitMove(){try{let j=await api('/api/submit',{room,pid:me});say(j.message);await refresh()}catch(e){say(e.message)}}
async function undoMove(){try{await api('/api/undo',{room,pid:me});selected=null;await refresh()}catch(e){say(e.message)}}
async function passTurn(){try{let j=await api('/api/pass',{room,pid:me});say(j.message);await refresh()}catch(e){say(e.message)}}
async function shareLink(){let url=location.origin+'/?room='+room;try{await navigator.clipboard.writeText(url);say('Посилання скопійовано! Надішли його другу.')}catch(e){prompt('Скопіюй посилання:',url)}}
const params=new URLSearchParams(location.search);if(params.get('room')){$('code').value=params.get('room');$('lobbymsg').textContent='Введи ім’я та натисни «Приєднатися».'}
</script></body></html>"""

@app.get("/")
def home():
    return render_template_string(HTML)

@app.get("/health")
def health():
    return {"ok": True, "dictionary": dict_status}

@app.post("/api/create")
def create():
    data=request.get_json(force=True)
    code=''.join(random.choices(string.ascii_uppercase+string.digits,k=5))
    pid=0
    with lock:
        room={"code":code,"players":[{"name":str(data.get("name","Гравець 1"))[:18],"score":0,"rack":[],"joined":True},
                                     {"name":"","score":0,"rack":[],"joined":False}],
              "board":[[None for _ in range(15)] for _ in range(15)],"bag":new_bag(),"turn":0,
              "pending":[[],[]],"started":False,"status":"Очікуємо другого гравця","last_message":"Кімнату створено. Надішли посилання другу."}
        for i in range(2): draw_tiles(room,i)
        rooms[code]=room
    return jsonify(code=code,pid=pid)

@app.post("/api/join")
def join():
    data=request.get_json(force=True); code=str(data.get("code","")).upper()
    with lock:
        room=rooms.get(code)
        if not room: return jsonify(error="Кімнату не знайдено. Перевір код або посилання."),404
        if room["players"][1]["joined"]: return jsonify(error="У цій кімнаті вже є два гравці."),409
        room["players"][1].update(name=str(data.get("name","Гравець 2"))[:18],joined=True)
        room["started"]=True;room["last_message"]="Друг приєднався. Гра почалася!"
    return jsonify(code=code,pid=1)

@app.get("/api/state/<code>/<int:pid>")
def get_state(code,pid):
    with lock:
        room=rooms.get(code)
        if not room or pid not in (0,1): return jsonify(error="Кімнату не знайдено."),404
        return jsonify(snapshot(room,pid))

@app.post("/api/place")
def place():
    d=request.get_json(force=True);code=d.get("room");pid=d.get("pid")
    with lock:
        room=rooms.get(code)
        if not room:return jsonify(error="Кімнату не знайдено."),404
        if not room["started"]:return jsonify(error="Очікуємо другого гравця."),400
        if pid!=room["turn"]:return jsonify(error="Зараз не твій хід."),400
        r,c,i=int(d["r"]),int(d["c"]),int(d["index"])
        if not(0<=r<15 and 0<=c<15 and 0<=i<len(room["players"][pid]["rack"])):return jsonify(error="Некоректна клітинка або літера."),400
        if room["board"][r][c] is not None:return jsonify(error="Клітинка зайнята."),400
        letter=room["players"][pid]["rack"].pop(i)
        room["board"][r][c]={"l":letter,"p":POINTS[letter],"new":True}
        room["pending"][pid].append([r,c])
        room["last_message"]="Літеру поставлено."
    return jsonify(ok=True)

@app.post("/api/undo")
def undo():
    d=request.get_json(force=True);code=d.get("room");pid=d.get("pid")
    with lock:
        room=rooms.get(code)
        if not room:return jsonify(error="Кімнату не знайдено."),404
        if pid!=room["turn"]:return jsonify(error="Зараз не твій хід."),400
        for r,c in room["pending"][pid]:
            t=room["board"][r][c]
            if t:room["players"][pid]["rack"].append(t["l"]);room["board"][r][c]=None
        room["pending"][pid]=[];room["last_message"]="Літери повернуто."
    return jsonify(ok=True)

@app.post("/api/pass")
def passed():
    d=request.get_json(force=True);code=d.get("room");pid=d.get("pid")
    with lock:
        room=rooms.get(code)
        if not room:return jsonify(error="Кімнату не знайдено."),404
        if pid!=room["turn"]:return jsonify(error="Зараз не твій хід."),400
        if room["pending"][pid]:return jsonify(error="Спочатку поверни непідтверджені літери."),400
        room["turn"]=1-pid;room["last_message"]=room["players"][pid]["name"]+" пропустив хід."
    return jsonify(ok=True,message="Хід пропущено.")

@app.post("/api/submit")
def submit():
    d=request.get_json(force=True);code=d.get("room");pid=d.get("pid")
    with lock:
        room=rooms.get(code)
        if not room:return jsonify(error="Кімнату не знайдено."),404
        if pid!=room["turn"]:return jsonify(error="Зараз не твій хід."),400
        pending=room["pending"][pid]
        if not pending:return jsonify(error="Спочатку постав літери."),400
        if len(pending)>1:
            rs={x[0] for x in pending};cs={x[1] for x in pending}
            if len(rs)>1 and len(cs)>1:return jsonify(error="Нові літери мають бути в одному рядку або стовпці."),400
            if len(rs)==1:
                r=next(iter(rs)); cmin=min(x[1] for x in pending);cmax=max(x[1] for x in pending)
                if any(room["board"][r][c] is None for c in range(cmin,cmax+1)):return jsonify(error="Між літерами не повинно бути порожніх клітинок."),400
            else:
                c=next(iter(cs));rmin=min(x[0] for x in pending);rmax=max(x[0] for x in pending)
                if any(room["board"][r][c] is None for r in range(rmin,rmax+1)):return jsonify(error="Між літерами не повинно бути порожніх клітинок."),400
        occupied_before=[(r,c) for r in range(15) for c in range(15) if room["board"][r][c] and not room["board"][r][c].get("new")]
        if not occupied_before:
            if not any(r==7 and c==7 for r,c in pending):return jsonify(error="Перше слово має проходити через центральну клітинку ★."),400
        else:
            adjacent=False
            for r,c in pending:
                for dr,dc in ((0,1),(0,-1),(1,0),(-1,0)):
                    rr,cc=r+dr,c+dc
                    if 0<=rr<15 and 0<=cc<15 and room["board"][rr][cc] and not room["board"][rr][cc].get("new"):adjacent=True
            if not adjacent:return jsonify(error="Нове слово має торкатися вже викладених літер."),400
        # Identify main direction; include each cross-word.
        if len({r for r,c in pending})==1:dr,dc=0,1
        elif len({c for r,c in pending})==1:dr,dc=1,0
        else:
            r,c=pending[0]
            horizontal=(c>0 and room["board"][r][c-1]) or (c<14 and room["board"][r][c+1])
            dr,dc=(0,1) if horizontal else (1,0)
        words=[]
        w,cells=get_line(room["board"],pending[0][0],pending[0][1],dr,dc)
        if len(w)>1:
            if not all(tuple(x) in cells for x in pending):return jsonify(error="Нові літери мають утворювати одне суцільне слово."),400
            words.append((w,cells))
        cr,cc=dc,dr
        for r,c in pending:
            w,cells=get_line(room["board"],r,c,cr,cc)
            if len(w)>1:words.append((w,cells))
        if not words:return jsonify(error="Потрібно утворити слово щонайменше з двох літер."),400
        bad=[w for w,_ in words if norm(w) not in dictionary]
        if bad:return jsonify(error="Немає у словнику: "+", ".join(bad)+". Перевір написання."),400
        points=sum(room["board"][r][c]["p"] for _,cells in words for r,c in cells)
        if len(pending)==7:points+=50
        room["players"][pid]["score"]+=points
        for r,c in pending:room["board"][r][c]["new"]=False
        room["pending"][pid]=[]
        draw_tiles(room,pid)
        room["turn"]=1-pid
        room["last_message"]=f"{room['players'][pid]['name']} склав: {', '.join(w for w,_ in words)} (+{points})."
    return jsonify(ok=True,message=room["last_message"])

if __name__=="__main__":
    app.run(host="0.0.0.0",port=PORT)
