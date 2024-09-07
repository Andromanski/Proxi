import requests
from bs4 import BeautifulSoup
import time

# Список сайтов с SOCKS5 прокси
proxy_sites = [
    'https://www.socks-proxy.net/',
    'https://www.proxy-list.download/SOCKS5',
    'https://spys.one/en/socks-proxy-list/',
    'https://www.proxy-listen.de/Proxy/Proxyliste.html',
    'https://www.my-proxy.com/free-socks-5-proxy.html',
    'https://www.socks-proxy.net/',
    'https://proxylist.geonode.com/',
    'https://www.free-proxy-list.com/socks5',
    'https://openproxy.space/list/socks5',
    'https://hidemy.name/en/proxy-list/?type=5',
    'https://www.proxydocker.com/en/proxylist/country/SOCKS5',
    'https://www.proxyscan.io/',
    'https://free-proxy-list.net/socks5-proxy.html',
    'https://www.proxyserverlist24.top/',
    'https://www.sslproxies.org/',
    'https://list.proxylistplus.com/SOCKS-List-1',
    'https://proxypedia.org/en/proxy-lists/socks5/',
    'https://www.ipaddress.com/proxy-list/',
    'https://geonode.com/free-proxy-list/',
    'https://www.proxy-listen.de/Proxy/Socks5-Proxy-List.html'
]

# Таймаут для проверки прокси
TIMEOUT = 5  # секунды

# Лимит времени ответа (медленные прокси отбрасываются)
MAX_RESPONSE_TIME = 3  # секунды

# Функция для парсинга прокси с сайтов
def get_proxies_from_site(url, table_id=None, table_class=None):
    proxies = []
    try:
        response = requests.get(url)
        soup = BeautifulSoup(response.content, 'lxml')

        # Поиск таблицы с прокси
        if table_id:
            proxy_table = soup.find(id=table_id)
        elif table_class:
            proxy_table = soup.find('table', class_=table_class)
        else:
            proxy_table = soup.find('table')

        if not proxy_table:
            print(f"No table found on {url}")
            return proxies
        
        # Проходим по каждой строке таблицы и извлекаем IP и порт
        for row in proxy_table.tbody.find_all('tr'):
            cols = row.find_all('td')
            if len(cols) >= 2:
                ip = cols[0].text.strip()
                port = cols[1].text.strip()
                proxy = f"{ip}:{port}"
                proxies.append(proxy)

    except requests.exceptions.RequestException as e:
        print(f"Failed to fetch proxies from {url}: {e}")
    
    return proxies

# Проверка работоспособности SOCKS5 прокси
def check_proxy_socks5(proxy):
    try:
        proxies = {
            'http': f'socks5://{proxy}',
            'https': f'socks5://{proxy}'
        }
        start_time = time.time()
        response = requests.get('https://httpbin.org/ip', proxies=proxies, timeout=TIMEOUT)
        response_time = time.time() - start_time

        if response.status_code == 200 and response_time <= MAX_RESPONSE_TIME:
            return True, response_time
        else:
            return False, response_time
    except requests.exceptions.RequestException:
        return False, None

# Собираем прокси и проверяем их
def gather_proxies():
    all_proxies = []
    for site in proxy_sites:
        print(f"Fetching proxies from: {site}")
        proxies = get_proxies_from_site(site)
        all_proxies.extend(proxies)
    
    # Проверяем каждый прокси
    good_proxies = []
    for proxy in all_proxies:
        print(f"Checking proxy: {proxy}")
        is_good, response_time = check_proxy_socks5(proxy)
        if is_good:
            print(f"Good proxy: {proxy} (Response time: {response_time:.2f} seconds)")
            good_proxies.append(proxy)
        else:
            print(f"Bad or slow proxy: {proxy}")
    
    return good_proxies

# Сохраняем хорошие прокси в файл
def save_proxies_to_file(proxies, filename='good_socks5_proxies.txt'):
    with open(filename, 'w') as file:
        for proxy in proxies:
            file.write(f"{proxy}\n")
    print(f"Saved {len(proxies)} good proxies to {filename}")

# Основной блок
if __name__ == "__main__":
    good_proxies = gather_proxies()
    if good_proxies:
        save_proxies_to_file(good_proxies)
    else:
        print("No good proxies found.")
