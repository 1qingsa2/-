"""
百度翻译爬虫（Selenium 驱动真实浏览器）


"""

import csv
import time

from selenium import webdriver
from selenium.common.exceptions import TimeoutException, WebDriverException
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.edge.options import Options
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait

HOME = 'https://fanyi.baidu.com/mtpe-individual/multimodal'
INPUT_SEL = 'div[contenteditable="true"]'
# 译文区容器（注意区分原文区 div.scroll-y）
RESULT_SEL = 'div.scroll-y-dst span.sentId'
# 取译文的 JS：把译文区所有句子拼起来
READ_JS = """
const nodes = document.querySelectorAll('div.scroll-y-dst span.sentId');
return Array.from(nodes).map(n => (n.textContent || '').trim()).filter(Boolean).join('');
"""
# 普通 Chrome UA。无头模式的 UA 带 "HeadlessChrome"，会被百度直接识破，必须换掉。
UA = ('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 '
      '(KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36')


class BaiduFanyi:
    """百度翻译。用法：

        with BaiduFanyi() as fanyi:
            print(fanyi.translate('hello'))          # 英 -> 中
            print(fanyi.translate('你好', to='en'))  # 中 -> 英
    """

    def __init__(self, headless=False, timeout=25):
        opts = Options()
        if headless:
            opts.add_argument('--headless=new')
        opts.add_argument('--disable-gpu')
        opts.add_argument('--window-size=1400,900')
        # 关键：换掉无头默认的 HeadlessChrome UA，否则百度不发 Acs-Token
        opts.add_argument(f'--user-agent={UA}')
        # 去掉「正受到自动测试软件的控制」提示，降低被判定为爬虫的概率
        opts.add_experimental_option('excludeSwitches', ['enable-automation'])
        opts.add_experimental_option('useAutomationExtension', False)

        self.timeout = timeout
        self.driver = webdriver.Edge(options=opts)
        self.driver.set_script_timeout(timeout)
        self._lang = None

    # ---------- 内部方法 ----------

    def _ensure_lang(self, from_, to):
        """语言方向变了才重新进页面（方向由 URL 决定，改一次刷一次）。"""
        if self._lang == (from_, to):
            return
        url = f'{HOME}?lang={from_}2{to}'
        self.driver.get(url)
        WebDriverWait(self.driver, self.timeout).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, INPUT_SEL)))
        time.sleep(1.5)          # 等前端把编辑器挂载好
        self._lang = (from_, to)

    def _type(self, text):
        """往输入框里写文本（先清空）。"""
        box = self.driver.find_element(By.CSS_SELECTOR, INPUT_SEL)
        box.click()
        time.sleep(0.2)
        box.send_keys(Keys.CONTROL, 'a')
        box.send_keys(Keys.DELETE)
        time.sleep(0.2)
        # 逐字符输入，避免 React 受控组件丢字符
        for ch in text:
            box.send_keys(ch)
        time.sleep(0.4)

    def _wait_result(self, old):
        """等译文区出现新结果（和上一次不同且非空）。"""
        deadline = time.time() + self.timeout
        while time.time() < deadline:
            try:
                now = self.driver.execute_script(READ_JS)
            except WebDriverException:
                now = ''
            if now and now != old:
                return now
            time.sleep(0.3)
        raise TimeoutException(f'等待译文超时，最后一次拿到的是: {old!r}')

    # ---------- 对外方法 ----------

    def translate(self, text, from_='en', to='zh'):
        """翻译单条文本，返回译文。"""
        self._ensure_lang(from_, to)
        old = self.driver.execute_script(READ_JS)
        self._type(text)
        return self._wait_result(old)

    def translate_batch(self, texts, from_='en', to='zh'):
        """批量翻译，返回 [(原文, 译文), ...]。"""
        self._ensure_lang(from_, to)
        out = []
        old = self.driver.execute_script(READ_JS)
        for text in texts:
            self._type(text)
            result = self._wait_result(old)
            old = result
            out.append((text, result))
        return out

    def close(self):
        if self.driver:
            self.driver.quit()
            self.driver = None

    def __enter__(self):
        return self

    def __exit__(self, *exc):
        self.close()


def save_csv(pairs, path='fanyi.csv'):
    with open(path, 'w', encoding='utf-8-sig', newline='') as f:
        w = csv.writer(f)
        w.writerow(['原文', '译文'])
        w.writerows(pairs)
    print(f'已写入 {path}')

fx=input('请输入')
if __name__ == '__main__':
    words = [f'{fx}']

    with BaiduFanyi(headless=False) as fanyi:      # 想看过程就留着 False
        for src, dst in fanyi.translate_batch(words):
            print(f'{src}  ->  {dst}')