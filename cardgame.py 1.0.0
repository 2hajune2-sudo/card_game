#!/usr/bin/env python3
"""한 장 승부 - 턴제 카드 배틀 (터미널 / AI 대전 / 온라인 대전)

실행:
    python cardgame.py                  메뉴
    python cardgame.py host [포트]      방 만들기 (기본 포트 5050)
    python cardgame.py join IP [포트]   방 참가
    python cardgame.py --no-color       색 끄기

카드와 규칙 숫자는 cards.json 에서 바꿉니다. 새 효과를 만들려면
OPS(문구)와 Game.run_fx(동작)에 추가하세요.
표준 라이브러리만 사용하므로 별도 설치가 필요 없습니다.
"""
import json
import os
import random
import socket
import sys
import time

HERE = os.path.dirname(os.path.abspath(__file__))
DECK_FILE = os.path.join(HERE, "deck.json")
DEFAULT_PORT = 5050

# ----------------------------------------------------------------------
# 데이터 로드
# ----------------------------------------------------------------------
def _discard_text(e):
    return "상대 손패 전부 버리기" if e["v"] >= 99 else f"상대 손패 {e['v']}장 버리기"


def _dmgif_text(e):
    who = "내" if e["who"] == "me" else "상대"
    cmp_ = "이하" if e["cmp"] == "le" else "이상"
    return f"상대에게 {e['v']} 피해 ({who} 체력 {e['t']} {cmp_}면 +{e['b']})"


def fx_text(fx):
    return ", ".join(OPS[e["op"]](e) for e in fx)


def _chance_text(e):
    win = fx_text(e["win"])
    lose = fx_text(e.get("lose", []))
    if lose:
        return f"{e['p']}% 확률로 [{win}], 아니면 [{lose}]"
    return f"{e['p']}% 확률로 [{win}]"


# 효과 문구: 효과 dict -> 설명. 새 효과를 만들면 여기와 Game.run_fx 에 추가하세요.
OPS = {
    "dmg":          lambda e: f"상대에게 {e['v']} 피해",
    "selfdmg":      lambda e: f"나에게 {e['v']} 피해",
    "heal":         lambda e: f"체력 {e['v']} 회복",
    "draw":         lambda e: f"카드 {e['v']}장 뽑기",
    "discard":      _discard_text,
    "discardSelf":  lambda e: f"내 손패 {e['v']}장 버리기",
    "breakInstall": lambda e: f"상대 설치 {e['v']}개 파괴",
    "steal":        lambda e: f"상대 손패 {e['v']}장 훔치기",
    "poison":       lambda e: f"상대를 {e['v']}턴 중독 (턴 시작 1 피해)",
    "shield":       lambda e: f"보호막 {e['v']} (다음 피해 흡수)",
    "swapHp":       lambda e: "상대와 체력 맞바꾸기",
    "handSwap":     lambda e: "상대와 손패 통째로 맞바꾸기",
    "timeStop":     lambda e: "상대의 다음 턴 건너뛰기",
    "dmgPerCopy":   lambda e: f"손패의 같은 카드 1장당 {e['v']} 피해 (이 카드 포함)",
    "healPerCopy":  lambda e: f"손패의 같은 카드 1장당 체력 {e['v']} 회복 (이 카드 포함)",
    "dmgIf":        _dmgif_text,
    "dmgRand":      lambda e: f"상대에게 {e['v']}~{e['b']} 피해 (무작위)",
    "healRand":     lambda e: f"체력 {e['v']}~{e['b']} 회복 (무작위)",
    "dmgPerInstall":  lambda e: f"내 설치 카드 1장당 상대에게 {e['v']} 피해",
    "healPerInstall": lambda e: f"내 설치 카드 1장당 체력 {e['v']} 회복",
    "chance":       _chance_text,
}
ON_LABEL = {"play": "설치 즉시", "turnStart": "내 턴 시작",
            "turnEnd": "내 턴 끝", "hit": "피해를 받을 때", "expire": "시한 만료"}
RN = {"common": "일반", "rare": "레어", "hero": "영웅", "legend": "레전드"}
KN = {"instant": "즉발", "install": "설치"}
RORDER = ["common", "rare", "hero", "legend"]
RCOLOR = {"common": "37", "rare": "34", "hero": "35", "legend": "33"}


def load_data():
    path = os.path.join(HERE, "cards.json")
    with open(path, encoding="utf-8") as f:
        data = json.load(f)
    cfg = data["config"]
    cards = {}
    for c in data["cards"]:
        if c["id"] in cards:
            sys.exit(f"cards.json 오류: id 중복 {c['id']}")
        if c["rarity"] not in RN or c["kind"] not in KN:
            sys.exit(f"cards.json 오류: {c['id']} 의 rarity/kind 가 올바르지 않아요")
        def check(fx):
            for e in fx:
                if e["op"] not in OPS:
                    sys.exit(f"cards.json 오류: {c['id']} 의 알 수 없는 효과 {e['op']}")
                if e["op"] == "chance":
                    check(e["win"])
                    check(e.get("lose", []))
        check(c.get("fx", []))
        if c.get("on") == "expire" and not c.get("duration"):
            sys.exit(f"cards.json 오류: {c['id']} 는 on=expire 이므로 duration(몇 턴 뒤)이 필요해요")
        cards[c["id"]] = c
    limit = {r: (v if v is not None else 10 ** 9) for r, v in cfg["rarity_limit"].items()}
    return cfg, cards, limit


CFG, CARDS, LIMIT = load_data()
COLOR = True


def paint(text, code):
    return f"\033[{code}m{text}\033[0m" if COLOR else text


def describe(c):
    fx = fx_text(c.get("fx", []))
    if c["kind"] == "instant":
        return fx + "."
    parts = []
    m = c.get("mods", {})
    if m.get("dealBonus"):
        parts.append(f"내 즉발 피해 +{m['dealBonus']}")
    tr = m.get("takeReduce", 0)
    if tr > 0:
        parts.append(f"받는 피해 -{tr}")
    elif tr < 0:
        parts.append(f"받는 피해 +{-tr}")
    if m.get("surviveOnce"):
        parts.append("체력이 0이 되면 1로 버티고 사라짐")
    if c.get("on") == "expire":
        parts.append(f"설치 {c['duration']}턴 뒤 발동: {fx}")
        return " · ".join(parts)
    if fx:
        need = f"설치 {c['need']}장 이상일 때 " if c.get("need") else ""
        parts.append(f"{need}{ON_LABEL[c.get('on', 'play')]}: {fx}")
    if c.get("duration"):
        parts.append(f"{c['duration']}턴 지속")
    else:
        parts.append("1회용" if m.get("surviveOnce") else "영구")
    return " · ".join(parts)


def card_line(cid):
    c = CARDS[cid]
    tag = paint(f"[{RN[c['rarity']]}·{KN[c['kind']]}] {c['name']}", RCOLOR[c["rarity"]])
    return f"{tag} — {describe(c)}"


# ----------------------------------------------------------------------
# 덱 규칙
# ----------------------------------------------------------------------
def rarity_counts(deck):
    n = {r: 0 for r in RORDER}
    for cid in deck:
        if cid in CARDS:
            n[CARDS[cid]["rarity"]] += 1
    return n


def validate_deck(deck):
    if not isinstance(deck, list) or not CFG["deck_min"] <= len(deck) <= CFG["deck_max"]:
        return (f"덱은 {CFG['deck_min']}~{CFG['deck_max']}장이어야 해요. "
                f"(지금 {len(deck) if isinstance(deck, list) else '?'}장)")
    if any(cid not in CARDS for cid in deck):
        return "존재하지 않는 카드가 있어요."
    n = rarity_counts(deck)
    for r in RORDER:
        if n[r] > LIMIT[r]:
            return f"{RN[r]} 카드는 최대 {LIMIT[r]}장이에요."
    return None


def auto_deck():
    """등급 제한 안에서 균형 잡힌 20장 덱을 무작위로 만듭니다 (AI 상대 덱 / 자동 구성용)."""
    size = CFG["deck_max"]
    pools = {r: [c["id"] for c in CARDS.values() if c["rarity"] == r] for r in RORDER}
    want = {"legend": min(LIMIT["legend"], 2), "hero": min(LIMIT["hero"], random.randint(5, 6)),
            "rare": min(LIMIT["rare"], random.randint(9, 10))}
    deck = []
    for r in ("legend", "hero", "rare"):
        if pools[r]:
            deck += [random.choice(pools[r]) for _ in range(want[r])]
    while len(deck) < size:
        deck.append(random.choice(pools["common"]))
    return random.sample(deck[:size], size)


def load_deck():
    try:
        with open(DECK_FILE, encoding="utf-8") as f:
            return [c for c in json.load(f) if c in CARDS]
    except (OSError, ValueError):
        return auto_deck()


def save_deck(deck):
    try:
        with open(DECK_FILE, "w", encoding="utf-8") as f:
            json.dump(deck, f)
    except OSError:
        pass


# ----------------------------------------------------------------------
# 게임 엔진 (화면과 완전히 분리)
# ----------------------------------------------------------------------
class Player:
    def __init__(self, deck):
        self.hp = CFG["start_hp"]
        self.deck_list = list(deck)
        self.deck = random.sample(deck, len(deck))
        self.hand = []
        self.field = []   # {"id", "left", "fresh"}
        self.poison = 0   # 남은 중독 턴
        self.shield = 0   # 남은 보호막
        self.skip = 0     # 건너뛸 턴 수

    def mod(self, key):
        return sum(CARDS[f["id"]].get("mods", {}).get(key, 0) for f in self.field)


class Game:
    def __init__(self, deck0, deck1):
        self.p = [Player(deck0), Player(deck1)]
        self.turn = random.randint(0, 1)
        self.over = False
        self.winner = None  # 0, 1, -1(무승부)
        self.log = []
        self.seq = 0
        self.installed = False  # 이번 턴에 설치 카드를 썼는지 (턴당 1장)
        self.last = None        # 마지막으로 사용된 카드 (플레이어, 카드 id)
        for pi in (0, 1):
            for _ in range(CFG["start_hand"]):
                self.take_card(pi)
        self.say(self.turn, "선공으로 시작해요")
        self.start_turn(first=True)

    # --- 기록 / 뽑기 ---
    def say(self, who, text):
        self.log.append((who, text))
        del self.log[:-30]

    def take_card(self, pi):
        p = self.p[pi]
        if not p.deck:
            return
        cid = p.deck.pop()
        if len(p.hand) >= CFG["hand_limit"]:
            self.say(pi, f"손패가 가득 차 {CARDS[cid]['name']} 소멸")
        else:
            p.hand.append(cid)

    def draw(self, pi, n):
        p = self.p[pi]
        for _ in range(n):
            if not p.deck:  # 덱을 다 뽑았다 → 리필 후 refill_draw 장
                p.deck = random.sample(p.deck_list, len(p.deck_list))
                self.say(pi, f"덱을 {len(p.deck_list)}장으로 다시 채웠어요")
                for _ in range(CFG["refill_draw"]):
                    self.take_card(pi)
            else:
                self.take_card(pi)

    # --- 전투 ---
    def check_over(self):
        for i in (0, 1):
            P = self.p[i]
            if P.hp <= 0:  # 불굴의 맹세 같은 '버티기' 설치
                for f in P.field:
                    if CARDS[f["id"]].get("mods", {}).get("surviveOnce"):
                        P.field.remove(f)
                        P.hp = 1
                        self.say(i, f"{CARDS[f['id']]['name']} 발동! 체력 1로 버텨요")
                        break
        dead = [i for i in (0, 1) if self.p[i].hp <= 0]
        if dead:
            self.over = True
            self.winner = -1 if len(dead) == 2 else 1 - dead[0]

    def fire(self, pi, trig, no_hit=False):
        """pi 의 설치 카드 중 trig 시점에 발동하는 것을 실행 (need: 설치 장수 조건)."""
        P = self.p[pi]
        for f in list(P.field):
            d = CARDS[f["id"]]
            if d["kind"] == "install" and d.get("on") == trig and len(P.field) >= d.get("need", 0):
                self.run_fx(pi, d["fx"], no_hit=no_hit)
                if self.over:
                    return

    def deal_damage(self, tgt, amt, no_hit=False):
        if amt <= 0:
            return
        P = self.p[tgt]
        red = P.mod("takeReduce")
        real = max(1, amt - red) if red > 0 else amt - red   # 방어로 줄어도 피해는 최소 1
        notes = []
        if red > 0:
            notes.append(f"방어 -{min(red, amt)}")
        elif red < 0:
            notes.append(f"취약 +{-red}")
        if P.shield > 0 and real > 0:
            ab = min(P.shield, real)
            P.shield -= ab
            real -= ab
            notes.append(f"보호막 -{ab}")
        P.hp -= real
        extra = f" ({', '.join(notes)})" if notes else ""
        self.say(tgt, f"{real} 피해{extra} · 체력 {max(P.hp, 0)}")
        self.check_over()
        if real > 0 and not no_hit and not self.over:
            self.fire(tgt, "hit", no_hit=True)

    def run_fx(self, o, fx, bonus=False, no_hit=False, cid=None):
        P, Q = self.p[o], self.p[1 - o]
        copies = P.hand.count(cid) + 1 if cid else 1   # 손패의 같은 카드 수 (사용한 카드 포함)
        extra = P.mod("dealBonus") if bonus else 0
        for e in fx:
            if self.over:
                break
            op = e["op"]
            v = e.get("v", 0)
            if op == "dmg":
                self.deal_damage(1 - o, v + extra, no_hit)
            elif op == "dmgPerCopy":
                self.deal_damage(1 - o, v * copies + extra, no_hit)
            elif op == "dmgIf":
                who = o if e["who"] == "me" else 1 - o
                hp = self.p[who].hp
                cond = hp <= e["t"] if e["cmp"] == "le" else hp >= e["t"]
                self.deal_damage(1 - o, v + (e["b"] if cond else 0) + extra, no_hit)
            elif op == "dmgRand":
                n = random.randint(v, e["b"])
                if n + extra <= 0:
                    self.say(o, "주사위가 빗나갔어요")
                else:
                    self.deal_damage(1 - o, n + extra, no_hit)
            elif op == "dmgPerInstall":
                self.deal_damage(1 - o, v * len(P.field) + extra, no_hit)
            elif op == "healRand":
                n = random.randint(v, e["b"])
                P.hp = min(CFG["max_hp"], P.hp + n)
                self.say(o, f"체력 +{n} · 체력 {P.hp}")
            elif op == "healPerInstall":
                n = v * len(P.field)
                P.hp = min(CFG["max_hp"], P.hp + n)
                self.say(o, f"체력 +{n} · 체력 {P.hp}")
            elif op == "chance":
                win = random.random() * 100 < e["p"]
                self.say(o, "행운! 🍀" if win else "불운… 💧")
                self.run_fx(o, e["win"] if win else e.get("lose", []), bonus, no_hit, cid)
            elif op == "selfdmg":
                P.hp -= v
                self.say(o, f"스스로 {v} 피해 · 체력 {max(P.hp, 0)}")
                self.check_over()
            elif op == "heal":
                P.hp = min(CFG["max_hp"], P.hp + v)
                self.say(o, f"체력 +{v} · 체력 {P.hp}")
            elif op == "healPerCopy":
                P.hp = min(CFG["max_hp"], P.hp + v * copies)
                self.say(o, f"체력 +{v * copies} · 체력 {P.hp}")
            elif op == "draw":
                self.draw(o, v)
                self.say(o, f"카드 {v}장 뽑기")
            elif op == "discard":
                n = 0
                for _ in range(v):
                    if Q.hand:
                        Q.hand.pop(random.randrange(len(Q.hand)))
                        n += 1
                self.say(1 - o, f"손패 {n}장 버려짐")
            elif op == "discardSelf":
                n = 0
                for _ in range(v):
                    if P.hand:
                        P.hand.pop(random.randrange(len(P.hand)))
                        n += 1
                self.say(o, f"손패 {n}장 버림")
            elif op == "steal":
                n = 0
                for _ in range(v):
                    if Q.hand:
                        c = Q.hand.pop(random.randrange(len(Q.hand)))
                        if len(P.hand) < CFG["hand_limit"]:
                            P.hand.append(c)
                        n += 1
                self.say(o, f"상대 손패 {n}장 훔침")
            elif op == "breakInstall":
                for _ in range(v):
                    if Q.field:
                        f = Q.field.pop(random.randrange(len(Q.field)))
                        self.say(1 - o, f"{CARDS[f['id']]['name']} 파괴됨")
            elif op == "poison":
                Q.poison = max(Q.poison, v)
                self.say(1 - o, f"중독! ({Q.poison}턴)")
            elif op == "shield":
                P.shield += v
                self.say(o, f"보호막 {P.shield}")
            elif op == "swapHp":
                P.hp, Q.hp = Q.hp, P.hp
                self.say(o, f"체력 교환! 나 {P.hp} · 상대 {Q.hp}")
                self.check_over()
            elif op == "handSwap":
                P.hand, Q.hand = Q.hand, P.hand
                self.say(o, "손패를 통째로 맞바꿨어요")
            elif op == "timeStop":
                Q.skip += v
                self.say(o, "시간 정지! 상대의 다음 턴을 건너뛰어요")

    # --- 턴 진행 ---
    def start_turn(self, first=False):
        pi = self.turn
        P = self.p[pi]
        self.installed = False
        if P.poison > 0:
            P.poison -= 1
            self.say(pi, "중독 피해")
            self.deal_damage(pi, 1, no_hit=True)
            if self.over:
                return
        self.fire(pi, "turnStart")
        if self.over:
            return
        if not (first and CFG["first_player_skips_draw"]):
            self.draw(pi, CFG["draw_per_turn"])

    def pass_turn(self):
        cur = self.turn
        P = self.p[cur]
        self.fire(cur, "turnEnd")
        if self.over:
            return
        keep, expired = [], []
        for f in P.field:
            if f["fresh"]:
                f["fresh"] = False
                keep.append(f)
            elif f["left"] is None:
                keep.append(f)
            else:
                f["left"] -= 1
                if f["left"] > 0:
                    keep.append(f)
                elif CARDS[f["id"]].get("on") == "expire":
                    expired.append(f)
                else:
                    self.say(cur, f"{CARDS[f['id']]['name']} 효과 종료")
        P.field = keep
        for f in expired:   # 시한 설치: 정해진 턴이 지나면 발동하고 사라짐
            d = CARDS[f["id"]]
            self.say(cur, f"{d['name']} 발동!")
            self.run_fx(cur, d["fx"])
            if self.over:
                return
        nxt = 1 - cur
        if self.p[nxt].skip > 0:   # 시간 정지: 상대 턴을 건너뛰고 내가 다시 진행
            self.p[nxt].skip -= 1
            self.say(nxt, "멈춰 있어서 턴을 건너뛰어요")
            self.turn = cur
        else:
            self.turn = nxt
        self.start_turn()

    # --- 플레이어 행동: (성공 여부, 메시지) ---
    def play_card(self, pi, cid):
        if self.over or self.turn != pi:
            return False, "내 차례가 아니에요."
        p = self.p[pi]
        if cid not in p.hand:
            return False, "손패에 없는 카드예요."
        d = CARDS[cid]
        if d["kind"] == "install":
            if self.installed:
                return False, "설치 카드는 한 턴에 1장만 쓸 수 있어요."
            if len(p.field) >= CFG["field_max"]:
                return False, f"설치 칸이 가득 찼어요 (최대 {CFG['field_max']})."
            p.hand.remove(cid)
            self.last = (pi, cid)
            p.field.append({"id": cid, "left": d.get("duration") or None, "fresh": True})
            self.installed = True
            self.say(pi, f"{d['name']} 설치")
            if d.get("on") == "play" and len(p.field) >= d.get("need", 0):
                self.run_fx(pi, d["fx"])
        else:
            p.hand.remove(cid)
            self.last = (pi, cid)
            self.say(pi, f"{d['name']} 사용")
            self.run_fx(pi, d["fx"], bonus=True, cid=cid)
            if not self.over:
                self.pass_turn()
        self.seq += 1
        return True, ""

    def end_turn(self, pi):
        if self.over or self.turn != pi:
            return False, "내 차례가 아니에요."
        if any(CARDS[c]["kind"] == "instant" for c in self.p[pi].hand):
            return False, "손패에 즉발 카드가 있으면 턴을 넘길 수 없어요."
        self.last = None
        self.pass_turn()
        self.seq += 1
        return True, ""

    def act(self, pi, a):
        if a.get("a") == "play":
            return self.play_card(pi, a.get("id"))
        if a.get("a") == "end":
            return self.end_turn(pi)
        return False, "알 수 없는 행동이에요."

    # --- pi 시점의 화면 데이터 (상대 손패/덱 내용은 숨김, JSON 직렬화 가능) ---
    def view_for(self, pi):
        me, op = self.p[pi], self.p[1 - pi]
        fld = lambda p: [[f["id"], f["left"]] for f in p.field]
        if not self.over:
            w = None
        else:
            w = -1 if self.winner == -1 else (1 if self.winner == pi else 0)
        return {
            "q": self.seq, "t": 1 if self.turn == pi else 0, "o": 1 if self.over else 0, "w": w,
            "me": {"hp": me.hp, "hand": list(me.hand), "dn": len(me.deck), "f": fld(me),
                   "in": 1 if (self.turn == pi and self.installed) else 0,
                   "ps": me.poison, "sh": me.shield},
            "op": {"hp": op.hp, "hn": len(op.hand), "dn": len(op.deck), "f": fld(op),
                   "ps": op.poison, "sh": op.shield},
            "lp": None if not self.last else [0 if self.last[0] == pi else 1, self.last[1]],
            "log": [[-1 if x is None else (0 if x == pi else 1), t] for x, t in self.log[-6:]],
        }


# ----------------------------------------------------------------------
# AI
# ----------------------------------------------------------------------
def fx_value(fx, me, op, copies, bonus):
    sc = 0.0
    room = CFG["max_hp"] - me.hp
    reduce_ = op.mod("takeReduce")
    for e in fx:
        k, v = e["op"], e.get("v", 0)
        dmg = kill = None
        if k == "dmg":
            dmg = kill = v
        elif k == "dmgPerCopy":
            dmg = kill = v * copies
        elif k == "dmgPerInstall":
            dmg = kill = v * len(me.field)
        elif k == "dmgRand":
            dmg, kill = (v + e["b"]) / 2, v
        elif k == "dmgIf":
            hp = me.hp if e["who"] == "me" else op.hp
            dmg = kill = v + (e["b"] if (hp <= e["t"] if e["cmp"] == "le" else hp >= e["t"]) else 0)
        if dmg is not None:
            sc += (dmg + bonus) * 3
            if kill + bonus >= op.hp + reduce_:
                sc += 100
        elif k in ("heal", "healPerCopy", "healPerInstall", "healRand"):
            amt = {"heal": v, "healPerCopy": v * copies, "healPerInstall": v * len(me.field),
                   "healRand": (v + e.get("b", v)) / 2}[k]
            sc += min(amt, room) * (4 if me.hp <= 6 else 0.8)   # 체력이 위험할 때만 회복을 선호
        elif k == "chance":
            p = e["p"] / 100
            sc += p * fx_value(e["win"], me, op, copies, bonus) + (1 - p) * fx_value(e.get("lose", []), me, op, copies, bonus)
        elif k == "draw":
            sc += v * 1.5
        elif k == "discard":
            sc += min(len(op.hand), v) * 2
        elif k == "discardSelf":
            sc -= min(len(me.hand) - 1, v) * 2
        elif k == "steal":
            sc += min(len(op.hand), v) * 3
        elif k == "breakInstall":
            sc += 4 * v if op.field else 0
        elif k == "poison":
            sc += 2.5 * v if op.poison < v else 0
        elif k == "shield":
            sc += v * 1.5
        elif k == "selfdmg":
            sc -= v * 3 + (1000 if me.hp <= v else 0)
        elif k == "swapHp":
            sc += (op.hp - me.hp) * 3 - 3
        elif k == "handSwap":
            sc += (len(op.hand) - len(me.hand)) * 2 - 2
        elif k == "timeStop":
            sc += 7
    return sc


def ai_score(d, me, op):
    return fx_value(d["fx"], me, op, me.hand.count(d["id"]), me.mod("dealBonus")) + random.random()


def ai_choose(g, pi):
    me, op = g.p[pi], g.p[1 - pi]
    installs = [c for c in me.hand if CARDS[c]["kind"] == "install"]
    if installs and not g.installed and len(me.field) < CFG["field_max"]:
        return {"a": "play", "id": installs[0]}
    instants = [c for c in me.hand if CARDS[c]["kind"] == "instant"]
    if instants:
        best = max(instants, key=lambda c: ai_score(CARDS[c], me, op))
        return {"a": "play", "id": best}
    return {"a": "end"}


def simple_choose(view):
    """뷰 데이터만으로 고르는 간단한 자동 플레이 (테스트용 --auto)"""
    hand = view["me"]["hand"]
    if not view["me"].get("in") and len(view["me"]["f"]) < CFG["field_max"]:
        for c in hand:
            if CARDS[c]["kind"] == "install":
                return {"a": "play", "id": c}
    for c in hand:
        if CARDS[c]["kind"] == "instant":
            return {"a": "play", "id": c}
    return {"a": "end"}


# ----------------------------------------------------------------------
# 터미널 화면
# ----------------------------------------------------------------------
AUTO = False


def bar(hp):
    n = max(0, min(hp, CFG["max_hp"]))
    color = "31" if hp <= 5 else "32"
    return paint("█" * n, color) + "░" * (CFG["max_hp"] - n)


def field_text(f):
    if not f:
        return "없음"
    out = []
    for cid, left in f:
        c = CARDS[cid]
        out.append(paint(f"{c['name']}" + (f"({left}턴)" if left else "(영구)"), RCOLOR[c["rarity"]]))
    return "  ".join(out)


def status_text(side):
    s = ""
    if side.get("ps"):
        s += f"  ☠중독 {side['ps']}"
    if side.get("sh"):
        s += f"  🛡보호막 {side['sh']}"
    return s


def render(v):
    line = "═" * 60
    print("\n" + line)
    print(f"상대  체력 {max(v['op']['hp'], 0):>2}  {bar(v['op']['hp'])}   손패 {v['op']['hn']} · 덱 {v['op']['dn']}{status_text(v['op'])}")
    print(f"  설치: {field_text(v['op']['f'])}")
    print("─" * 60)
    for who, text in v["log"][-5:]:
        print(("  " if who < 0 else ("  나: " if who == 0 else "  상대: ")) + text)
    print("─" * 60)
    print(f"나    체력 {max(v['me']['hp'], 0):>2}  {bar(v['me']['hp'])}   덱 {v['me']['dn']}{status_text(v['me'])}")
    print(f"  설치: {field_text(v['me']['f'])}")
    print("  손패:")
    if not v["me"]["hand"]:
        print("    (비었어요)")
    for i, cid in enumerate(v["me"]["hand"], 1):
        print(f"   {i:>2}) {card_line(cid)}")
    print(line)


def ask_action(view):
    """내 턴의 입력을 받아 행동(dict)을 돌려줍니다. 종료하려면 SystemExit."""
    if AUTO:
        return simple_choose(view)
    hand = view["me"]["hand"]
    has_instant = any(CARDS[c]["kind"] == "instant" for c in hand)
    if view["me"].get("in"):
        hint = "이번 턴 설치는 이미 했어요. 즉발 카드를 쓰거나 턴을 넘기세요."
    elif has_instant:
        hint = "설치 카드는 턴당 1장, 즉발 카드를 쓰면 턴이 끝나요."
    else:
        hint = "즉발 카드가 없어요. 설치를 마쳤다면 e 로 턴을 넘기세요."
    print(f"내 차례 — {hint}")
    while True:
        s = input("번호=카드 사용 / e=턴 넘기기 / q=나가기 > ").strip().lower()
        if s == "q":
            raise SystemExit
        if s == "e":
            return {"a": "end"}
        if s.isdigit() and 1 <= int(s) <= len(hand):
            return {"a": "play", "id": hand[int(s) - 1]}
        print("올바른 번호나 명령을 입력해 주세요.")


def show_result(v):
    word = {1: "승리! 🎉", 0: "패배…", -1: "무승부"}[v["w"]]
    print("\n" + "★" * 60 + f"\n  게임 종료 — {word}\n" + "★" * 60)


# ----------------------------------------------------------------------
# AI 대전
# ----------------------------------------------------------------------
def play_ai(deck):
    g = Game(deck, auto_deck())
    seen = len(g.log)
    while not g.over:
        if g.turn == 0:
            v = g.view_for(0)
            render(v)
            ok, msg = g.act(0, ask_action(v))
            if not ok:
                print("⚠", msg)
            seen = len(g.log)
        else:
            time.sleep(0 if AUTO else 0.8)
            g.act(1, ai_choose(g, 1))
            for who, t in g.log[seen:]:
                print(f"  {'상대' if who == 1 else '나'}: {t}")
            seen = len(g.log)
    render(g.view_for(0))
    show_result(g.view_for(0))


# ----------------------------------------------------------------------
# 온라인 (TCP, 줄 단위 JSON). 방장이 게임을 진행하고 참가자는 행동만 보냅니다.
# ----------------------------------------------------------------------
class Conn:
    def __init__(self, sock):
        self.sock = sock
        self.f = sock.makefile("rw", encoding="utf-8", newline="\n")

    def send(self, obj):
        self.f.write(json.dumps(obj, ensure_ascii=False) + "\n")
        self.f.flush()

    def recv(self):
        line = self.f.readline()
        return json.loads(line) if line else None

    def close(self):
        try:
            self.sock.close()
        except OSError:
            pass


def local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except OSError:
        return "127.0.0.1"


def host_game(deck, port):
    srv = socket.socket()
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("", port))
    srv.listen(1)
    print(f"방을 열었어요. 상대에게 알려 주세요 → python cardgame.py join {local_ip()} {port}")
    print("(같은 와이파이/LAN이면 위 주소, 인터넷이면 포트 포워딩이 필요해요) 기다리는 중…")
    sock, addr = srv.accept()
    srv.close()
    c = Conn(sock)
    hello = c.recv()
    err = validate_hello(hello)
    if err:
        c.send({"t": "err", "msg": err})
        c.close()
        print("상대의 덱이 올바르지 않아 종료해요:", err)
        return
    print(f"{addr[0]} 접속! 게임을 시작합니다.")
    g = Game(deck, hello["deck"])
    try:
        c.send({"t": "st", "v": g.view_for(1), "n": None})
        while not g.over:
            if g.turn == 0:
                v = g.view_for(0)
                render(v)
                ok, msg = g.act(0, ask_action(v))
                if not ok:
                    print("⚠", msg)
                    continue
                c.send({"t": "st", "v": g.view_for(1), "n": None})
            else:
                render(g.view_for(0))
                print("상대 차례… 기다리는 중")
                m = c.recv()
                if m is None:
                    print("상대의 연결이 끊어졌어요.")
                    return
                ok, msg = g.act(1, m)
                c.send({"t": "st", "v": g.view_for(1), "n": None if ok else msg})
        render(g.view_for(0))
        show_result(g.view_for(0))
    except (BrokenPipeError, ConnectionError):
        print("상대의 연결이 끊어졌어요.")
    finally:
        c.close()


def validate_hello(m):
    if not m or m.get("t") != "hello":
        return "잘못된 접속이에요."
    return validate_deck(m.get("deck"))


def join_game(deck, ip, port):
    try:
        c = Conn(socket.create_connection((ip, port), timeout=10))
    except OSError as e:
        print("접속하지 못했어요:", e)
        return
    c.sock.settimeout(None)
    c.send({"t": "hello", "deck": deck})
    try:
        while True:
            m = c.recv()
            if m is None:
                print("방장의 연결이 끊어졌어요.")
                return
            if m["t"] == "err":
                print("거절됨:", m["msg"])
                return
            if m.get("n"):
                print("⚠", m["n"])
            v = m["v"]
            render(v)
            if v["o"]:
                show_result(v)
                return
            if v["t"] == 1:
                c.send(ask_action(v))
            else:
                print("상대 차례… 기다리는 중")
    except (BrokenPipeError, ConnectionError):
        print("방장의 연결이 끊어졌어요.")
    finally:
        c.close()


# ----------------------------------------------------------------------
# 덱 구성 / 규칙 / 메뉴
# ----------------------------------------------------------------------
def deck_menu(deck):
    while True:
        n = rarity_counts(deck)
        meters = "  ".join(
            f"{RN[r]} {n[r]}/{'∞' if LIMIT[r] >= 10 ** 9 else LIMIT[r]}" for r in RORDER)
        print(f"\n덱 {len(deck)}장 (최소 {CFG['deck_min']} ~ 최대 {CFG['deck_max']})   {meters}")
        for cid, c in CARDS.items():
            cnt = deck.count(cid)
            print(f"  {cid:<4} {card_line(cid)}   {paint(f'x{cnt}', '1') if cnt else ''}")
        print("명령: +ID [장수]  -ID [장수]  auto(자동)  clear(비우기)  done(완료)")
        s = input("덱 > ").strip().lower().split()
        if not s:
            continue
        cmd = s[0]
        if cmd == "done":
            err = validate_deck(deck)
            if err:
                print("⚠", err, "(그래도 저장은 돼요. 대전 전에 고쳐 주세요)")
            save_deck(deck)
            return deck
        if cmd == "auto":
            deck = auto_deck()
        elif cmd == "clear":
            deck = []
        elif cmd[0] in "+-" and cmd[1:] in CARDS:
            cid = cmd[1:]
            times = int(s[1]) if len(s) > 1 and s[1].isdigit() else 1
            for _ in range(times):
                if cmd[0] == "+":
                    if len(deck) >= CFG["deck_max"]:
                        print("⚠ 덱이 가득 찼어요.")
                        break
                    r = CARDS[cid]["rarity"]
                    if rarity_counts(deck)[r] >= LIMIT[r]:
                        print(f"⚠ {RN[r]} 카드는 최대 {LIMIT[r]}장이에요.")
                        break
                    deck.append(cid)
                elif cid in deck:
                    deck.remove(cid)
        else:
            print("알 수 없는 명령이에요.")
        save_deck(deck)


def rules_text():
    lim = CFG["rarity_limit"]
    return (
        f"■ 덱: {CFG['deck_min']}~{CFG['deck_max']}장. 일반 제한 없음 / 레어 {lim['rare']} / 영웅 {lim['hero']} / 레전드 {lim['legend']}장까지\n"
        f"■ 시작 체력 {CFG['start_hp']}, 손패 {CFG['start_hand']}장, 선공 무작위, 체력 0이면 패배\n"
        f"■ 설치 카드: 턴을 쓰지 않지만 한 턴에 1장만 (설치 칸 최대 {CFG['field_max']})\n"
        "■ 즉발 카드: 사용하면 턴이 끝나요. 손패에 즉발 카드가 있으면 턴을 넘길 수 없어요\n"
        f"■ 매 턴 시작에 {CFG['draw_per_turn']}장 뽑기{' (선공 첫 턴 제외)' if CFG['first_player_skips_draw'] else ''}\n"
        f"■ 덱을 다 뽑았다면 내 덱 전체로 리필하고 {CFG['refill_draw']}장 뽑기\n"
        f"■ 손패 최대 {CFG['hand_limit']}장 (넘치면 뽑은 카드는 사라져요)\n"
        "■ 설치 중에는 'N턴 뒤 발동'(시한 만료)이나 '설치가 N장 이상일 때만 발동'하는 카드가 있어요\n"
        "■ 확률 카드는 결과가 무작위예요 (방장 컴퓨터가 굴려요)\n"
        "■ 받는 피해 감소(방어)로는 피해를 1 아래로 줄일 수 없어요 (보호막은 전부 막을 수 있어요)\n"
        "■ 등급: 일반(수치 1, 같은 카드를 모으면 강해짐) / 레어(2~3, 쓸만한 특수) /\n"
        "        영웅(3, 강하지만 리스크) / 레전드(4~5, 판을 뒤집는 특수 능력)")


def show_rules():
    print("\n" + rules_text())


def need_valid(deck):
    err = validate_deck(deck)
    if err:
        print("⚠", err, "→ 덱 구성에서 고쳐 주세요.")
        return False
    return True


def menu():
    deck = load_deck()
    while True:
        print("\n===== 한 장 승부 =====\n 1) AI 대전\n 2) 방 만들기 (온라인)\n 3) 방 참가 (온라인)\n 4) 덱 구성\n 5) 규칙\n 0) 종료")
        s = input("선택 > ").strip()
        if s == "1" and need_valid(deck):
            play_ai(deck)
        elif s == "2" and need_valid(deck):
            p = input(f"포트 (엔터={DEFAULT_PORT}) > ").strip()
            host_game(deck, int(p) if p.isdigit() else DEFAULT_PORT)
        elif s == "3" and need_valid(deck):
            ip = input("방장 IP > ").strip()
            p = input(f"포트 (엔터={DEFAULT_PORT}) > ").strip()
            join_game(deck, ip, int(p) if p.isdigit() else DEFAULT_PORT)
        elif s == "4":
            deck = deck_menu(deck)
        elif s == "5":
            show_rules()
        elif s == "0":
            return


def main(argv):
    global COLOR, AUTO
    if "--no-color" in argv or not sys.stdout.isatty():
        COLOR = False
    if "--auto" in argv:
        AUTO = True
    args = [a for a in argv if not a.startswith("--")]
    if os.name == "nt":
        os.system("")  # Windows 터미널에서 색상 활성화
    try:
        if not args:
            menu()
        elif args[0] == "host":
            deck = load_deck() if not AUTO else auto_deck()
            if need_valid(deck):
                host_game(deck, int(args[1]) if len(args) > 1 else DEFAULT_PORT)
        elif args[0] == "join" and len(args) > 1:
            deck = load_deck() if not AUTO else auto_deck()
            if need_valid(deck):
                join_game(deck, args[1], int(args[2]) if len(args) > 2 else DEFAULT_PORT)
        elif args[0] == "ai":
            deck = load_deck() if not AUTO else auto_deck()
            if need_valid(deck):
                play_ai(deck)
        else:
            print(__doc__)
    except (KeyboardInterrupt, EOFError, SystemExit):
        print("\n게임을 종료합니다.")


if __name__ == "__main__":
    main(sys.argv[1:])
