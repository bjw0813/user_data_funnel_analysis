# 이커머스 로그데이터 - 퍼널분석

깃허브 자리

# 데이터

![image.png](image.png)

- event_time: 이벤트가 발생한 시각
- event_type: 이벤트 종류
    - view: 상품을 조회
    - cart: 상품을 카트에 추가
    - remove_from_cart: 상품을 카트에서 제거
    - purchase: 구매
- product_id: 상품번호
- category_id: 카테고리번호
- category_code: 카테고리명
- brand: 브랜드명
- price: 상품 가격
- user_id: 고객번호
- user_session: 세션

**데이터 출처: kaggle**

# 분석 목표(분석 전 질문)

- DAU(일간 활성 사용자수) 추이는?
    - 어느 요일에 가장 많이 방문하는지
    
- 고객들의 사이트 체류시간 평균은?
    - 조회만 한 유저, 카트에 담은 유저, 구매까지 한 유저별로 체류시간이 어떻게 다른지
    
- 세 단계(조회, 카트, 구매) 중 어느 단계에서 유저 이탈이 가장 많이 발생하나?

# 데이터 전처리

```python
data["event_time"]=pd.to_datetime(data["event_time"])
```

```python
data.isna().sum()
```

![image.png](image%201.png)

결측값이 너무 많아 `category_id`와 `category_code`는 drop하기로 결정

```python
data.drop(["category_code","brand"],axis=1,inplace=True)
```

```python
data["date_ymd"]=data["event_time"].dt.date
```

DAU(일간 활성 사용자 수)를 위해 date_ymd라는 필드를 따로 만들어줬다

![image.png](image%202.png)

# 데이터 분석

## DAU(일간 활성 사용자수) 추이

```python
dau=data.groupby("date_ymd")[["user_id"]].nunique().reset_index().rename({"user_id":"dau"},axis=1)
dau.head(10)
```

![image.png](image%203.png)

```python
plt.figure(figsize=(12,5))
a=sns.lineplot(x="date_ymd",y="dau",data=dau)

plt.title("DAU")
plt.show()
```

![image.png](image%204.png)

```python
import plotly.express as px 
px.line(data_frame=dau,x="date_ymd",y="dau")
```

![image.png](image%205.png)

```python
dau["date_ymd"]=pd.to_datetime(dau["date_ymd"])
dau["dayname"]=dau["date_ymd"].dt.day_name()
avg_dau=dau.groupby("dayname")[["dau"]].mean()
avg_dau.reset_index(inplace=True)
px.bar(data_frame=avg_dau,x="dayname",y="dau")#화요일에 활성이용자수가 가장 높고, 주말이 가장 적다.
```

![image.png](image%206.png)

## 고객들의 사이트 체류시간 평균은?

## 세 단계(조회, 카트, 구매) 중 어느 단계에서 유저 이탈이 가장 많이 발생하나?

```python
funnel=pd.DataFrame(pivot[["view","cart","remove_from_cart","purchase"]].sum()).reset_index()
funnel
funnel=funnel[funnel["event_type"] != "remove_from_cart"]
funnel[0][3]
px.funnel(data_frame=funnel, x="event_type", y=0) 

```

![image.png](image%207.png)

**카트에서 구매단계로 넘어가는 과정에서 가장 큰 이탈이 발생하는 것을 확인할 수 있다!**

# 분석 결과

DAU(일간 활성 사용자수)

- 월 초에서 중순까지 DAU가 증가하다가, 이후 유지
- 화요일에 가장 많이 방문하고, 주말에 사용자수가 줄어든다.

사이트 체류시간 평균은?

- 체류시간 평균은 약 1시간
- 조회만 한 유저는 약 40분, 카트에 담은 유저는 약 2시간 40분, 구매까지 한 유저는 약 6시간 40분을 체류한다.

퍼널 분석

- 상품 조회를 한 후 카트를 담는 단계에서 약 47.3%만 남고
- 카트를 담고 구매를 하는 단계에서 약 9%만 남는다.
- 카트를 담고 구매를 하는 단계에서 이탈이 많이 일어나므로, 해당 단계에서 전환율을 높이기 위한 전략이 필요 → ***결제과정이 불편한지, 회원가입이 까다로운지 등을 체크하고 장바구니 쿠폰이나 장바구니 이벤트 등을 통해 구매 전환율을 높일 수 있는 방안을 고려할 필요가 있다!***

# 느낀점

로그데이터를 경험해보며 plotly를 처음 경험해봤는데 그래프가 굉장히 깔끔하게 나와서 앞으로도 자주 애용할 것 같고, 이번 경험을 통해 퍼널분석을 좀 더 자세하게 이해할 수 있었던 경험이었다.
