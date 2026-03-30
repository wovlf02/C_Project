# 칼로리 계산 및 센서 설계

> 최종 수정: 2026-03-30

---

## 1. 칼로리 계산 공식

```
소모 칼로리 = calorie_per_min × elapsed_min × intensity
```

| 변수 | 설명 | 범위 |
|------|------|------|
| calorie_per_min | 기구별 분당 기본 칼로리 (DB) | 3.00 ~ 10.00 |
| elapsed_min | 경과 시간 (분) | 누적 |
| intensity | 강도 계수 (센서 기반) | 0.5 ~ 2.0 |

---

## 2. 강도 계수 산출

```c
// intensity = min(2.0, max(0.5, sensor_value / base_value))
double calc_intensity(int sensor_value, int base_value) {
    double ratio = (double)sensor_value / base_value;
    if (ratio < 0.5) return 0.5;
    if (ratio > 2.0) return 2.0;
    return ratio;
}
```

### 기구별 base_value

| 기구 유형 | 센서 | base_value |
|-----------|------|------------|
| 트레드밀 | speed (km/h) | 8.0 |
| 실내자전거 | rpm | 60 |
| 일립티컬 | rpm | 50 |
| 로잉머신 | rpm | 30 |
| 근력 기구 | resistance | 10 |
| 스트레칭 | - | 1.0 (고정) |

---

## 3. 실시간 칼로리 갱신

```
매 HEARTBEAT (3초):
  1. 현재 elapsed_sec 계산
  2. 센서값으로 intensity 산출
  3. avg_intensity 누적 평균 계산
  4. current_calories = calorie_per_min × (elapsed_sec/60) × avg_intensity
  5. DASHBOARD 응답에 포함
```

---

## 4. 자동 로그아웃 시 칼로리 정산

- end_time = last_sensor_change (마지막 활동 시점)
- 비활동 구간 칼로리 제외
- duration_sec = last_sensor_change - start_time
- calories = calorie_per_min × (duration_sec/60) × avg_intensity
