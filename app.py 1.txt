import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go
from datetime import datetime, timedelta

# 1. 웹페이지 기본 설정
st.set_page_config(page_title="My Personal Stock Dashboard", layout="wide", initial_sidebar_state="expanded")

st.title("📈 나만의 맞춤형 주식 대시보드")
st.markdown("내가 선택한 기업들의 실시간 주가, 속보 및 장·단기 전망을 한눈에 확인하세요.")
st.write(f"최근 업데이트: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

# 2. 사이드바 - 관심 종목 관리
st.sidebar.header("📌 관심 종목 설정")

# 세션 상태를 이용해 종목 리스트 유지 (기본값: 빅테크 및 반도체 주요 기업)
if 'tickers' not in st.session_state:
    st.session_state.tickers = ["NVDA", "AAPL", "MSFT", "TSLA"]

# 종목 추가 UI
new_ticker = st.sidebar.text_input("추가할 Ticker 입력 (예: AMZN, GOOG):").upper().strip()
if st.sidebar.button("종목 추가") and new_ticker:
    if new_ticker not in st.session_state.tickers:
        st.session_state.tickers.append(new_ticker)
        st.rerun()
    else:
        st.sidebar.warning("이미 등록된 종목입니다.")

# 종목 삭제 UI
ticker_to_remove = st.sidebar.selectbox("삭제할 종목 선택:", ["선택 안 함"] + st.session_state.tickers)
if st.sidebar.button("종목 삭제") and ticker_to_remove != "선택 안 함":
    st.session_state.tickers.remove(ticker_to_remove)
    st.rerun()

# 3. 메인 화면 - 종목 선택
selected_ticker = st.selectbox("🎯 분석할 기업을 선택하세요:", st.session_state.tickers)

if selected_ticker:
    try:
        # 데이터 가져오기
        stock = yf.Ticker(selected_ticker)
        info = stock.info
        
        # 주가 데이터 (최근 1년)
        hist = stock.history(period="1y")
        
        if hist.empty:
            st.error("주가 데이터를 불러올 수 없습니다. 티커를 확인해주세요.")
        else:
            current_price = hist['Close'].iloc[-1]
            prev_close = hist['Close'].iloc[-2] if len(hist) > 1 else current_price
            price_change = current_price - prev_close
            price_change_pct = (price_change / prev_close) * 100
            
            # --- 섹션 1: 실시간 주가 및 요약 주요 지표 ---
            st.subheader(f"🏢 {info.get('longName', selected_ticker)} ({selected_ticker})")
            
            col1, col2, col3, col4 = st.columns(4)
            col1.metric("현재 주가", f"${current_price:,.2f}", f"{price_change_pct:+.2f}%")
            col2.metric("시가총액", f"${info.get('marketCap', 0):,}")
            col3.metric("52주 최고가", f"${info.get('fiftyTwoWeekHigh', 0):,.2f}")
            col4.metric("52주 최저가", f"${info.get('fiftyTwoWeekLow', 0):,.2f}")
            
            st.markdown("---")
            
            # --- 섹션 2: 장 / 단기 주가 전망 분석 ---
            st.subheader("🔮 장·단기 주가 전망 리포트")
            left_analysis, right_analysis = st.columns(2)
            
            with left_analysis:
                st.markdown("### ⏱️ 단기 전망 (기술적 지표 분석)")
                # 단기 지표 계산: RSI (Relative Strength Index)
                delta = hist['Close'].diff()
                gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
                loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
                rs = gain / loss
                rsi = 100 - (100 / (1 + rs)).iloc[-1]
                
                # 단기 이동평균선선 (20일)
                sma_20 = hist['Close'].rolling(window=20).mean().iloc[-1]
                
                st.write(f"**현재 RSI (14일):** {rsi:.2f}")
                if rsi > 70:
                    st.error("⚠️ **단기 과열 (과매수 상태):** 현재 단기적으로 주가가 급등하여 조정 가능성이 있습니다. 신규 진입은 신중해야 합니다.")
                elif rsi < 30:
                    st.success("✅ **단기 과매도 (저점 매수 기회):** 단기 낙폭이 과대하여 기술적 반등이 나올 수 있는 구간입니다.")
                else:
                    st.info("📊 **단기 중립:** 주가가 비교적 안정적인 흐름을 보이고 있으며, 뚜렷한 과매수/과매도 시그널은 없습니다.")
                
                st.write(f"**20일 이동평균선:** ${sma_20:,.2f} (현재가 대비 {'상회' if current_price > sma_20 else '하회'})")

            with right_analysis:
                st.markdown("### 🎯 장기 전망 (월가 목표 주가)")
                target_high = info.get('targetHighPrice')
                target_mean = info.get('targetMeanPrice')
                target_low = info.get('targetLowPrice')
                
                if target_mean:
                    upside = ((target_mean - current_price) / current_price) * 100
                    st.write(f"**기관 평균 목표가:** ${target_mean:,.2f}")
                    st.write(f"**목표 주가 대비 상승 여력:** {upside:+.2f}%")
                    
                    if upside > 15:
                        st.success(f"🚀 **장기 전망 밝음:** 월가 평균 목표가 대비 {upside:.1f}%의 추가 상승 여력이 있어 장기 투자 매력도가 높습니다.")
                    elif upside < 0:
                        st.error("⚠️ **장기 밸류에이션 부담:** 현재 주가가 기관들의 평균 장기 목표가를 넘어섰습니다. 기간 조정이나 매물 출회에 유의하세요.")
                    else:
                        st.info("🔄 **장기 적정 가치 구간:** 현재 주가가 기관들의 목표치와 유사하게 수렴해 있어, 펀더멘탈의 추가 변화를 주시해야 합니다.")
                else:
                    st.warning("해당 종목은 월가 목표 주가 데이터가 제공되지 않습니다.")

            st.markdown("---")

            # --- 섹션 3: 주가 차트 ---
            st.subheader("📈 주가 추이 (1년)")
            chart_type = st.radio("차트 형태 선택:", ["라인 차트", "캔들스틱 차트"], horizontal=True)
            
            fig = go.Figure()
            if chart_type == "라인 차트":
                fig.add_trace(go.Scatter(x=hist.index, y=hist['Close'], mode='lines', name='종가', line=dict(color='#00FFCC')))
            else:
                fig.add_trace(go.Candlestick(x=hist.index, open=hist['Open'], high=hist['High'], low=hist['Low'], close=hist['Close'], name='캔들'))
            
            fig.update_layout(template="plotly_dark", xaxis_rangeslider_visible=False, margin=dict(l=20, r=20, t=20, b=20))
            st.plotly_chart(fig, use_container_width=True)

            st.markdown("---")

            # --- 섹션 4: 기업 최신 뉴스 및 속보 ---
            st.subheader(f"📰 {selected_ticker} 관련 실시간 뉴스 & 속보")
            news_list = stock.news
            
            if news_list:
                for item in news_list[:5]:  # 최신 뉴스 5개 출력
                    title = item.get('title')
                    publisher = item.get('publisher', '알 수 없음')
                    link = item.get('link')
                    
                    # 뉴스 타임스탬프 변환
                    pub_time = item.get('providerPublishTime')
                    if pub_time:
                        date_str = datetime.fromtimestamp(pub_time).strftime('%Y-%m-%d %H:%M')
                    else:
                        date_str = ""
                        
                    st.markdown(f"**[{publisher}]** [{title}]({link})  *{date_str}*")
            else:
                st.write("최근 관련 뉴스가 없습니다.")

    except Exception as e:
        st.error(f"데이터를 불러오는 중 오류가 발생했습니다: {e}")