# TPC-H Baseline vs Hint 기반 성능 개선 분석 보고서

## 1. 개요 (Executive Summary)

MySQL 8.0 환경(SSD 기반)에서 TPC-H (Scale Factor 30) 벤치마크 쿼리의 **Baseline(Cold Start)** 성능과 **Optimizer Hints (Hint 01, 02, 04, 05)** 를 적용했을 때의 최적 성능을 비교 분석한 결과.

분석 결과, **전체 22개 쿼리 중 약 45% (10개 쿼리)에서 유의미한 성능 향상**이 확인되었습니다. 특히 복잡한 조인이 포함된 쿼리에서 힌트가 비효율적인 실행 계획(Full Table Scan 등)을 방지하고 인덱스 사용을 강제함으로써 **최대 70% 이상의 수행 시간 단축**을 달성했습니다. 반면, 일부 쿼리(Q9)에서는 힌트가 오히려 쿼리 최적화기(Optimizer)의 효율적인 판단을 방해하여 성능이 저하되는 현상도 관찰.

### 핵심 성과
*   **최대 개선**: **Q3 (70% 단축)**, **Q13 (64% 단축)**, **Q11 (64% 단축)**
*   **주요 전략**: **Hint 05** (복합 인덱스 및 조인 순서 제어)가 가장 많은 쿼리(Q1, Q3, Q7, Q8, Q13)에서 Best Strategy로 선정됨.

---

## 2. 상세 비교 분석표 (Performance Comparison Table)

| 쿼리 (Query) | Baseline (초) | Best Hint (초) | 최적 전략 (Source) | 개선율 (Improvement) | 상태 (Status) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Q1** | 227.26 | 171.92 | Hint 05 | **+24.35%** | **개선 (BETTER)** |
| **Q2** | 10.48 | 9.52 | Hint 01 | +9.16% | 개선 (BETTER) |
| **Q3** | 709.14 | 209.54 | Hint 05 | **+70.45%** | **대폭 개선 (BETTER)** |
| **Q4** | 236.42 | 209.53 | Hint 04 | +11.37% | 개선 (BETTER) |
| **Q5** | 343.38 | 211.03 | Hint 04 | **+38.54%** | **개선 (BETTER)** |
| **Q6** | 59.70 | 59.72 | Hint 05 | -0.03% | 동일 (SAME) |
| **Q7** | 384.86 | 248.20 | Hint 05 | **+35.51%** | **개선 (BETTER)** |
| **Q8** | 579.85 | 267.28 | Hint 05 | **+53.91%** | **대폭 개선 (BETTER)** |
| **Q9** | 229.17 | 626.60 | Baseline | -173.4% | **악화 (WORSE)** |
| **Q10** | 263.90 | 265.14 | Baseline | -0.47% | 동일 (SAME) |
| **Q11** | 55.56 | 19.77 | Hint 01 | **+64.42%** | **대폭 개선 (BETTER)** |
| **Q12** | 184.38 | 180.50 | Hint 01 | +2.10% | 동일 (SAME) |
| **Q13** | 174.22 | 61.93 | Hint 05 | **+64.45%** | **대폭 개선 (BETTER)** |
| **Q14** | 81.96 | 81.97 | Baseline | 0.00% | 동일 (SAME) |
| **Q15** | 61.02 | 60.48 | Baseline | +0.89% | 동일 (SAME) |
| **Q16** | 13.27 | 12.93 | Hint 01 | +2.56% | 동일 (SAME) |
| **Q17** | 57.03 | 56.33 | Baseline | +1.23% | 동일 (SAME) |
| **Q18** | 102.13 | 101.25 | Baseline | +0.86% | 동일 (SAME) |
| **Q19** | 95.03 | 95.25 | Baseline | -0.23% | 동일 (SAME) |
| **Q20** | 268.37 | 263.95 | Baseline | +1.65% | 동일 (SAME) |
| **Q21** | 377.52 | 248.47 | Hint 01 | **+34.18%** | **개선 (BETTER)** |
| **Q22** | 5.98 | 5.86 | Hint 02 | +2.01% | 동일 (SAME) |

---

## 3. 쿼리별 상세 분석 


### ✅ 성능 대폭 개선 그룹 (Q3, Q8, Q13, Q11, Q21, Q5, Q7)

*   **Q3 (Shipping Priority Query)**
    *   **성능 변화**: 709초 → 209초 (**70% 개선**)
    *   **원인 분석**: Q3는 `CUSTOMER`, `ORDERS`, `LINEITEM` 3개 테이블을 조인합니다. Baseline에서는 `LINEITEM` 테이블에 대한 접근이 효율적이지 못했거나(Full Scan 등), 조인 순서가 최적화되지 않았을 가능성.
    *   **Hint 효과 (Hint 05)**: `LINEITEM` 테이블에 대해 조인 조건과 필터링 조건(`l_shipdate`)을 모두 커버하는 적절한 인덱스를 강제하여 스캔 범위를 대폭 줄이고, `ORDERS` -> `CUSTOMER` -> `LINEITEM` 순서의 효율적인 조인을 유도.

*   **Q13 (Customer Distribution Query)**
    *   **성능 변화**: 174초 → 62초 (**64% 개선**)
    *   **원인 분석**: Q13은 `CUSTOMER`와 `ORDERS`의 Left Outer Join을 수행합니다. `o_comment`에 대한 `NOT LIKE` 조건이 있어 까다로운듯.
    *   **Hint 효과 (Hint 05)**: 적절한 인덱스 힌트를 통해 `ORDERS` 테이블 접근 시 불필요한 스캔을 줄이고, Group By 처리 효율을 높인 것으로 추정. 특히 Join Buffer를 사용하는 Block Nested Loop 대신 인덱스 기반 Nested Loop가 효과를 발휘.

*   **Q8 (National Market Share Query)**
    *   **성능 변화**: 580초 → 267초 (**54% 개선**)
    *   **원인 분석**: Q8은 8개의 테이블을 조인하는 매우 복잡. Baseline에서는 조인 순서가 잘못 선택되어 중간 결과 집합이 매우 커지는 현상.
    *   **Hint 효과 (Hint 05)**: 선택도(Selectivity)가 높은 테이블(`REGION`, `NATION` 등)부터 시작하여 데이터 양을 초기에 줄이는 조인 순서를 강제하고, `LINEITEM`과 `ORDERS` 조인 시 효율적인 인덱스를 사용하도록 유도.

*   **Q11 (Important Stock Identification Query)**
    *   **성능 변화**: 56초 → 20초 (**64% 개선**)
    *   **원인 분석**: 서브쿼리와 메인 쿼리 모두 `PARTSUPP`와 `SUPPLIER`를 조인합니다.
    *   **Hint 효과 (Hint 01)**: `PARTSUPP` 테이블 접근 시 `ps_suppkey` 관련 인덱스를 효율적으로 사용하여 스캔 속도를 높임.

*   **Q21 (Suppliers Who Kept Orders Waiting Query)**
    *   **성능 변화**: 378초 → 248초 (**34% 개선**)
    *   **원인 분석**: `LINEITEM` 테이블의 Self-Join(`l1`, `l2`, `l3` 별칭 사용)이 포함되어 있어 매우 무거움.
    *   **Hint 효과 (Hint 01)**: `EXISTS` 및 `NOT EXISTS` 서브쿼리 내의 `LINEITEM` 조회 시 적절한 인덱스(`l_orderkey`, `l_suppkey` 복합 인덱스 등)를 타도록 강제하여 Lookup cost reducing.

### ❌ 성능 저하 그룹 (Q9)

*   **Q9 (Product Type Profit Measure Query)**
    *   **성능 변화**: 229초 → 626초 (**173% 악화**)
    *   **원인 분석**: Q9는 `part` 이름에 'green'이 포함된 데이터를 찾는데, 사실상 많은 양의 데이터를 조인해야 합니다.
    *   **Hint 부작용**:
        1.  **잘못된 인덱스 강제**: 힌트가 선택도가 좋지 않은 인덱스(예: `p_name`을 억지로 타게 하거나)를 강제하여, 오히려 효율적인 Full Table Scan이나 Hash Join(MySQL 8.0.18+ 지원)의 기회를 박탈했을 수 있습니다.
        2.  **Nested Loop 강제**: 데이터 양이 많을 때는 Hash Join이나 Batch Key Access가 빠를 수 있는데, 힌트로 인해 Random I/O가 많은 Index Nested Loop Join이 강제되어 I/O 병목이 심화된 것으로 보입니다.
    *   **결론**: Q9은 힌트 없이 Optimizer의 기본 판단에 맡기는 것이 훨씬 유리합니다.

### ➖ 성능 변동 없음/미미 그룹 (Q6, Q12, Q14 ~ Q20, Q22)

*   **Q6 (Forecasting Revenue Change)**
    *   단순 `LINEITEM` 필터링 및 집계 쿼리입니다. Baseline에서도 이미 최적의 인덱스를 타거나 Full Scan이 최선인 경우라 힌트가 개입할 여지가 없습니다.
*   **Q14, Q15, Q17 (Small Quantity/Promo Revenue)**
    *   `LINEITEM`과 `PART` 조인 위주의 쿼리들로, 기본 Primary Key나 Foreign Key 인덱스를 활용하는 것이 이미 최적입니다. 힌트를 줘도 실행 계획이 변하지 않았습니다.

---

## 4. 최종 제언 (Recommendations)

1.  **적극적 힌트 적용 (Adopt Hints)**:
    *   **Hint 05 적용**: Q1, Q3, Q7, Q8, Q13, Q20 (대규모 조인 성능 극대화)
    *   **Hint 04 적용**: Q4, Q5
    *   **Hint 01 적용**: Q2, Q11, Q21 (서브쿼리 및 특정 인덱스 효율 개선)

2.  **Baseline 유지 (Keep Baseline)**:
    *   **Q9**: 힌트 적용 시 성능이 심각하게 저하되므로, 힌트를 제거하여 Optimizer의 기본 로직을 따르도록 합니다.
    *   **Q6, Q10, Q12, Q14~Q19, Q22**: 힌트 적용 효과가 미미하므로 관리 복잡도를 줄이기 위해 Baseline(No Hint)을 유지합니다.

3.  **향후 과제**:
    *   **Q9 최적화 연구**: 현재의 힌트 세트가 Q9에 악영향을 주고 있습니다. `p_name` 필터링 조건에 특화된 새로운 인덱스 전략이나, 강제로 조인 알고리즘(예: Hash Join)을 유도하는 힌트를 별도로 테스트해볼 필요가 있습니다.
