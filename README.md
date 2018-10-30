# api.py
import requests
from datetime import datetime
import plotly
import time


config = {
    'VK_ACCESS_TOKEN': 'e2909f04264c9a2415ebced5edf20f39c264df76536b6d9b78d47e1a8ba9a9b0d959146c1e2cf48a56aa3',
    'PLOTLY_USERNAME': 'nastya_sorokina',
    'PLOTLY_API_KEY': 'Gyly1dLqEiVKnomV9tes',
    'CD': 27,
    'CM': 10,
    'CY':2018
}
def get(url, params={}, timeout=1, max_retries=5, backoff_factor=0.3):
    """ Выполнить GET-запрос

    :param url: адрес, на который необходимо выполнить запрос
    :param params: параметры запроса
    :param timeout: максимальное время ожидания ответа от сервера
    :param max_retries: максимальное число повторных запросов
    :param backoff_factor: коэффициент экспоненциального нарастания задержки
    """ 
    for a in range(max_retries):
        response = requests.get(url, params = params, timeout = timeout)
        if response.status_code == 200:
            break
        backoff_delay = backoff_factor * (2 ** (a - 1))
        time.sleep(backoff_delay)
    if response.status_code != 200:
        print('Connection problem occurred')
        return None
    else:
        return response

def get_friends(user_id, fields='bdate'):
    """ Вернуть данных о друзьях пользователя

    :param user_id: идентификатор пользователя, список друзей которого нужно получить
    :param fields: список полей, которые нужно получить для каждого пользователя
    """
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert isinstance(fields, str), "fields must be string"
    assert user_id > 0, "user_id must be positive integer"
    access_token = '{VK_ACCESS_TOKEN}'.format(**config)
    domain = "https://api.vk.com/method"
    query_params = {
    'domain': domain,
    'access_token': access_token,
    'user_id': user_id,
    'fields': fields
    }
    query = "{domain}/friends.get?access_token={access_token}&user_id={user_id}&fields={fields}&v=5.53".format(**query_params)
    ans = get(query)
    if ans:
        return ans
    else:
        return None
def age_predict(user_id):
    """ Наивный прогноз возраста по возрасту друзей

    Возраст считается как медиана среди возраста всех друзей пользователя

    :param user_id: идентификатор пользователя
    """
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert user_id > 0, "user_id must be positive integer"
    rp = get_friends(user_id)
    if rp:
        count = 0
        age_sum = 0
        for e in range(len(rp.json()['response']['items'])):
            if 'bdate' in rp.json()['response']['items'][e]:
                if len(rp.json()['response']['items'][e]['bdate']) > 7:
                    age_sum += age(rp.json()['response']['items'][e]['bdate'])
                    count += 1
        age_pr = round(age_sum / count, 2)
        return age_pr
    else:
        return rp

def age(bdate):
    bdate = '.' + bdate
    year, bdate = date_number(bdate)
    month, bdate = date_number(bdate)
    day, _ = date_number(bdate)
    if month > config['CM']:
        age = config['CY'] - year - 1
    elif month == config['CM']:
        if day > config['CD']:
            age = config['CY'] - year - 1
        else:
            age = config['CY'] - year
    else:
        age = config['CY'] - year
    return age

def date_number(bdate):
    a = len(bdate) - 1
    num = 0
    b = 1
    while bdate[a] != '.':
        num += (ord(bdate[a]) - ord('0')) * b
        bdate = bdate[:a]
        b = b * 10
        a = a - 1
    bdate = bdate[:a]
    return num, bdate
    
def messages_get_history(user_id, count=20, offset=0):
    """ Получить историю переписки с указанным пользователем

    :param user_id: идентификатор пользователя, с которым нужно получить историю переписки
    :param offset: смещение в истории переписки
    :param count: число сообщений, которое нужно получить
    """
    assert isinstance(user_id, int), "user_id must be positive integer"
    assert user_id > 0, "user_id must be positive integer"
    assert isinstance(offset, int), "offset must be positive integer"
    assert offset >= 0, "user_id must be positive integer"
    assert count >= 0, "user_id must be positive integer"
    access_token = '{VK_ACCESS_TOKEN}'.format(**config)
    domain = "https://api.vk.com/method"
    query_params = {
    'domain': domain,
    'access_token': access_token,
    'offset': offset,
    'count': count,
    'user_id': user_id
    }
    query = "{domain}/messages.getHistory?access_token={access_token}&offset={offset}&count={count}&user_id={user_id}&v=5.53".format(**query_params)
    meslst = get(query).json()
    return meslst
def count_dates_from_messages(messages):
    """ Получить список дат и их частот

    :param messages: список сообщений
    """
    dalist = []
    freqlist = []
    for a in range(len(messages['response']['items'])):
        date = datetime.fromtimestamp(messages['response']['items'][a]['date']).strftime("%d-%m-%Y")
        if date in dalist:
            b = dalist.index(date)
            freqlist[b] += 1
        else:
            dalist.append(date)
            freqlist.append(1)
    return dalist, freqlist

def plotly_messages_freq(dalist, freqlist):
    """ Построение графика с помощью Plot.ly

    :param freq_list: список дат и их частот
    """
    username = '{PLOTLY_USERNAME}'.format(**config)
    api_key = '{PLOTLY_API_KEY}'.format(**config)
    plotly.tools.set_credentials_file(username=username, api_key=api_key)
    import plotly.plotly as py
    import plotly.graph_objs as go
    data = [go.Scatter(x=dalist,y=freqlist)]
    py.iplot(data)
    pass

def get_network(users_ids, as_edgelist=True):
    edgelist = []
    for i in users_ids:
        hfriends = get_friends(i, fields = 'sex').json()
        hfidlist = []
        for b in range(len(hfriends['response']['items'])):
            hfidlist.append(hfriends['response']['items'][b]['id'])
        for c in users_ids:
           if c in hfidlist:
                tup = (users_ids.index(i), users_ids.index(c))
                edgelist.append(tup)
    for d in range(len(edgelist) // 2):
        for e in range(len(edgelist)):
            if edgelist[d][0] == edgelist[e][1] and edgelist[d][1] == edgelist[e][0]:
                edgelist.pop(e)
                break
    plot_graph(edgelist)
    pass


def plot_graph(graph):
    import networkx as nx
    G = nx.Graph()
    G.add_edges_from(graph)
    nx.draw(G)
    pass

if __name__ == "__main__":
    meslst = messages_get_history(204749327, 350)
    dalist, freqlist = count_dates_from_messages(meslst)
    plotly_messages_freq(dalist, freqlist)'''
