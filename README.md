import dash
from dash import dcc, html, Input, Output, State, dash_table
import pandas as pd
import yfinance as yf
import plotly.graph_objs as go
import datetime

app = dash.Dash(__name__)
app.title = '股票價格與布林通道'

search_history = pd.DataFrame(columns=['股票代碼', '日期範圍'])

def get_stock_data(stock_code, date_range):
    try:
        df = yf.download(stock_code, start=date_range[0], end=date_range[1])
        if df.empty:
            return None
        # 補 NA
        df = df.fillna(method='ffill').fillna(method='bfill')
        df['Close'] = df['Close'].interpolate()
        df['MA20'] = df['Close'].rolling(20).mean()
        df['STD20'] = df['Close'].rolling(20).std()
        df['BB_up'] = df['MA20'] + 2*df['STD20']
        df['BB_dn'] = df['MA20'] - 2*df['STD20']
        df['Volume'] = df['Volume'].fillna(0)
        return df
    except Exception as e:
        print("get_stock_data error:", e)
        return None

app.layout = html.Div([
    html.H2("股票價格與布林通道"),
    html.Div([
        dcc.Input(id='stock_code', value='2330.TW', type='text', placeholder="輸入股票代碼"),
        dcc.DatePickerRange(
            id='date_range',
            min_date_allowed=datetime.date(2000, 1, 1),
            max_date_allowed=datetime.date.today(),
            start_date=datetime.date(2024, 1, 1),  # 切記不要預設未來日期
            end_date=datetime.date.today()
        ),
        html.Button("更新圖表", id='update', n_clicks=0)
    ]),
    html.Br(),
    html.Div(id='company_info'),
    html.Div(id='latest_price'),
    html.Div(id='today_range'),
    html.Div(id='tomorrow_buy'),
    html.Div(id='tomorrow_sell'),
    html.Div(id='this_week_range'),
    html.Div(id='next_week_high'),
    html.Div(id='next_week_low'),
    html.Div(id='this_month_high'),
    html.Div(id='this_month_low'),
    html.Div(id='this_month_range'),
    html.Div(id='error_message', style={'color': 'red'}),
    dcc.Graph(id='combined_plot'),
    html.Hr(),
    html.H4("搜尋歷史"),
    dash_table.DataTable(
        id='search_history',
        columns=[{'name': i, 'id': i} for i in search_history.columns],
        data=search_history.to_dict('records'),
        row_selectable='single',
        style_table={'overflowX': 'auto'}
    )
])

@app.callback(
    [Output('combined_plot', 'figure'),
     Output('company_info', 'children'),
     Output('latest_price', 'children'),
     Output('today_range', 'children'),
     Output('tomorrow_buy', 'children'),
     Output('tomorrow_sell', 'children'),
     Output('this_week_range', 'children'),
     Output('next_week_high', 'children'),
     Output('next_week_low', 'children'),
     Output('this_month_high', 'children'),
     Output('this_month_low', 'children'),
     Output('this_month_range', 'children'),
     Output('error_message', 'children'),
     Output('search_history', 'data')],
    [Input('update', 'n_clicks')],
    [State('stock_code', 'value'),
     State('date_range', 'start_date'),
     State('date_range', 'end_date'),
     State('search_history', 'data')]
)
def update_chart(n_clicks, stock_code, start_date, end_date, history_data):
    # 日期不能大於今天
    if not stock_code:
        return [go.Figure()] + [""]*11 + ["請輸入股票代碼", history_data or []]
    if pd.to_datetime(end_date) > datetime.datetime.today():
        end_date = datetime.datetime.today().strftime('%Y-%m-%d')
    df = get_stock_data(stock_code, [start_date, end_date])
    if df is None or df.empty:
        return [go.Figure()] + [""]*11 + ["無法取得股票資料，請確認股票代碼是否正確", history_data or []]

    latest_price = round(df['Close'].iloc[-1], 2)
    today_high = round(df['High'].iloc[-1], 2)
    today_low = round(df['Low'].iloc[-1], 2)
    today_range = round(abs(today_high - today_low), 2)
    tomorrow_buy = round(latest_price - today_range, 2)
    tomorrow_sell = round(latest_price + today_range, 2)
    last_week_range = round(df['Close'].tail(5).max() - df['Close'].tail(5).min(), 2)
    next_week_high = round(latest_price + last_week_range, 2)
    next_week_low = round(latest_price - last_week_range, 2)
    last_month_range = round(df['Close'].tail(20).max() - df['Close'].tail(20).min(), 2)
    this_month_high = round(latest_price + last_month_range, 2)
    this_month_low = round(latest_price - last_month_range, 2)

    info_text = f"股票代碼: {stock_code}"
    latest_price_text = f"最新股價: {latest_price}"
    today_range_text = f"今日波動範圍: {today_range}"
    tomorrow_buy_text = f"明日預測買點: {tomorrow_buy}"
    tomorrow_sell_text = f"明日預測賣點: {tomorrow_sell}"
    this_week_range_text = f"當周震幅: {last_week_range}"
    next_week_high_text = f"下周預測最高點: {next_week_high}"
    next_week_low_text = f"下周預測最低點: {next_week_low}"
    this_month_high_text = f"當月預測最高點: {this_month_high}"
    this_month_low_text = f"當月預測最低點: {this_month_low}"
    this_month_range_text = f"當月預測震幅: {last_month_range}"

    # 搜尋歷史紀錄更新
    new_entry = {'股票代碼': stock_code, '日期範圍': f"{start_date} 至 {end_date}"}
    history_df = pd.DataFrame(history_data or [], columns=['股票代碼', '日期範圍'])
    history_df = pd.concat([history_df, pd.DataFrame([new_entry])], ignore_index=True).drop_duplicates()
    history_list = history_df.to_dict('records')

    # 價格+布林通道圖
    price_fig = go.Figure()
    price_fig.add_trace(go.Scatter(x=df.index, y=df['Close'], mode='lines', name='收盤價', line=dict(color='black')))
    price_fig.add_trace(go.Scatter(x=df.index, y=df['BB_up'], mode='lines', name='布林上軌', line=dict(color='red')))
    price_fig.add_trace(go.Scatter(x=df.index, y=df['BB_dn'], mode='lines', name='布林下軌', line=dict(color='green')))
    price_fig.add_trace(go.Scatter(x=df.index, y=df['MA20'], mode='lines', name='布林中軌', line=dict(color='purple')))
    price_fig.add_trace(go.Scatter(x=[df.index[-1]+pd.Timedelta(days=1)], y=[tomorrow_buy], mode='markers', name='預測買點', marker=dict(color='green', size=10, symbol='triangle-up')))
    price_fig.add_trace(go.Scatter(x=[df.index[-1]+pd.Timedelta(days=1)], y=[tomorrow_sell], mode='markers', name='預測賣點', marker=dict(color='red', size=10, symbol='triangle-down')))
    # 畫水平線（改用 Scatter，保證各版 plotly 都能跑）
    for y, n, c in [
        (next_week_high, "下周高", 'lightcoral'),
        (next_week_low, "下周低", 'lightblue'),
        (this_month_high, "當月高", 'darkred'),
        (this_month_low, "當月低", 'darkblue')
    ]:
        price_fig.add_trace(go.Scatter(
            x=[df.index[0], df.index[-1]+pd.Timedelta(days=2)],
            y=[y, y], mode='lines', name=n, line=dict(color=c, dash='dash')
        ))
    price_fig.update_layout(title='股票價格與布林通道', xaxis_title='日期', yaxis_title='價格')

    # 成交量圖
    volume_fig = go.Figure()
    volume_fig.add_trace(go.Bar(x=df.index, y=df['Volume']/1000, name='成交量'))
    volume_fig.update_layout(title='成交量 (張數)', xaxis_title='日期', yaxis_title='成交量 (千張)')

    # 合併圖
    from plotly.subplots import make_subplots
    combined_fig = make_subplots(rows=2, cols=1, shared_xaxes=True, row_heights=[0.7, 0.3], vertical_spacing=0.08, subplot_titles=('收盤價與布林通道', '成交量'))
    for trace in price_fig.data:
        combined_fig.add_trace(trace, row=1, col=1)
    for trace in volume_fig.data:
        combined_fig.add_trace(trace, row=2, col=1)
    combined_fig.update_layout(height=700, showlegend=True)

    return [
        combined_fig,
        info_text,
        latest_price_text,
        today_range_text,
        tomorrow_buy_text,
        tomorrow_sell_text,
        this_week_range_text,
        next_week_high_text,
        next_week_low_text,
        this_month_high_text,
        this_month_low_text,
        this_month_range_text,
        "",
        history_list
    ]

import webbrowser
webbrowser.open("http://127.0.0.1:8050/")

if __name__ == '__main__':
    app.run(debug=True)
