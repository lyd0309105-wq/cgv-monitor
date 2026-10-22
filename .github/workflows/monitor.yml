import os, re, sys, time, datetime as dt
import requests
from playwright.sync_api import sync_playwright

WEBHOOK = os.getenv("DISCORD_WEBHOOK_URL") or "https://discord.com/api/webhooks/1562377055189082132/Av_7IRBBhTL5beuoy76nPJblnPEvRJQIZ4y_3_iKc2h_uuSIfKzXbQOM2iYpLLb2ZGa9"
KEYWORD = os.getenv("KEYWORD", "요루시카")
THEATER = os.getenv("THEATER", "홍대")
CUTOFF = dt.date.fromisoformat(os.getenv("CUTOFF_DATE", "2026-10-30"))  # 이 날짜 '이후(당일 포함)'
LOOP_MINUTES = int(os.getenv("LOOP_MINUTES", "50"))   # 한 번 실행 시 반복 시간
INTERVAL = int(os.getenv("INTERVAL_SEC", "120"))      # 2분
SEND_START = os.getenv("SEND_START_MESSAGE", "false") == "true"
KST = dt.timezone(dt.timedelta(hours=9))


def notify(msg):
    try:
        requests.post(WEBHOOK, json={"content": msg}, timeout=15)
    except Exception as e:
        print("discord error:", e)


def real_dates(day_numbers):
    """화면의 일(day) 숫자 목록 -> 실제 날짜(월 넘어감 처리)"""
    today = dt.datetime.now(KST).date()
    out, cur = [], None
    for d in day_numbers:
        if cur is None:
            y, m = today.year, today.month
            if d < today.day - 1:  # 이미 다음 달로 넘어간 경우
                m += 1
                if m == 13: y, m = y + 1, 1
            cur = dt.date(y, m, d)
        else:
            nxt = cur + dt.timedelta(days=1)
            if nxt.day == d:
                cur = nxt
            else:  # 월 경계
                y, m = (cur.year, cur.month + 1) if cur.month < 12 else (cur.year + 1, 1)
                cur = dt.date(y, m, d)
        out.append(cur)
    return out


def check(page):
    page.goto("https://cgv.co.kr", wait_until="domcontentloaded", timeout=60000)
    page.wait_for_timeout(3000)

    # 팝업 닫기 (있으면)
    for t in ["닫기", "오늘 하루 보지 않기"]:
        try: page.get_by_text(t).first.click(timeout=1500)
        except Exception: pass

    # 검색
    page.locator("button:has-text('검색'), a:has-text('검색'), [class*=search] button").first.click(timeout=10000)
    box = page.locator("input[type=text], input[type=search], input").first
    box.fill(KEYWORD)
    box.press("Enter")
    page.wait_for_timeout(3000)

    # 작품 들어가기
    page.get_by_text(re.compile(KEYWORD)).first.click(timeout=10000)
    page.wait_for_timeout(3000)

    # 예매하기
    page.get_by_text(re.compile("예매")).first.click(timeout=10000)
    page.wait_for_timeout(4000)

    # 지역/극장 선택 (홍대)
    try: page.get_by_text("서울", exact=True).first.click(timeout=3000)
    except Exception: pass
    page.get_by_text(re.compile(THEATER)).first.click(timeout=10000)
    page.wait_for_timeout(3000)

    # 날짜 수집 (활성화된 날짜만)  ※ 사이트 개편 시 이 셀렉터만 수정
    items = page.locator("[class*=date] li, [class*=date] button, [class*=day] li, [class*=day] button")
    nums, enabled = [], []
    for i in range(items.count()):
        el = items.nth(i)
        txt = (el.inner_text() or "").strip()
        m = re.search(r"\b(\d{1,2})\b", txt.replace("\n", " "))
        if not m: continue
        cls = (el.get_attribute("class") or "")
        dis = el.get_attribute("disabled") is not None or el.get_attribute("aria-disabled") == "true" \
              or re.search(r"disabled|dim|off|closed", cls)
        nums.append(int(m.group(1))); enabled.append(not dis)
    dates = real_dates(nums)
    return [d for d, ok in zip(dates, enabled) if ok and d >= CUTOFF]


def main():
    if SEND_START:
        notify(f"✅ CGV 모니터링 시작: '{KEYWORD}' / {THEATER} / {CUTOFF} 이후 날짜 감시 (2분 간격)")
    notified = set()
    end = time.time() + LOOP_MINUTES * 60
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        while time.time() < end:
            t0 = time.time()
            ctx = browser.new_context(locale="ko-KR", timezone_id="Asia/Seoul",
                viewport={"width": 1400, "height": 900},
                user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36")
            page = ctx.new_page()
            try:
                found = check(page)
                print(dt.datetime.now(KST), "found:", found)
                new = [d for d in found if d not in notified]
                if new:
                    notified.update(new)
                    notify(f"🎬 {KEYWORD} {THEATER} 예매 오픈! " + ", ".join(d.strftime("%m/%d(%a)") for d in new)
                           + "\nhttps://cgv.co.kr")
            except Exception as e:
                print("error:", e)
                page.screenshot(path="debug.png", full_page=True)
            finally:
                ctx.close()
            time.sleep(max(1, INTERVAL - (time.time() - t0)))
        browser.close()


if __name__ == "__main__":
    main()
