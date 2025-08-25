import random
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional, Tuple
import sys
import itertools
from collections import Counter

# (기존 코드와 동일한 부분)
class DiceRule(Enum):
    ONE = 0
    TWO = 1
    THREE = 2
    FOUR = 3
    FIVE = 4
    SIX = 5
    CHOICE = 6
    FOUR_OF_A_KIND = 7
    FULL_HOUSE = 8
    SMALL_STRAIGHT = 9
    LARGE_STRAIGHT = 10
    YACHT = 11

@dataclass
class Bid:
    group: str
    amount: int

@dataclass
class DicePut:
    rule: DiceRule
    dice: List[int]

class Game:
    def __init__(self):
        self.my_state = GameState()
        self.opp_state = GameState()
        self.round = 0
        self.opp_bid_history = []
        self.seen_dice_counts = Counter()
        self.contention_win_streak = 0
        self.contention_loss_streak = 0
        self.STORAGE_SIM_SAMPLES = 75

    def _calculate_immediate_score(self, dice: List[int], state: 'GameState') -> Tuple[int, Optional[DiceRule]]:
        best_score = -1
        best_rule = None
        num_dice_to_choose = min(5, len(dice))
        if num_dice_to_choose <= 0:
            return 0, None
        dice_combos = list(itertools.combinations(dice, num_dice_to_choose))
        for rule in state.get_available_rules():
            for combo in dice_combos:
                score = GameState.calculate_score(DicePut(rule, list(combo)))
                if score > best_score:
                    best_score = score
                    best_rule = rule
        return (best_score, best_rule) if best_score > -1 else (0, None)

    def _calculate_option_value(self, dice: List[int], state: 'GameState') -> float:
        if not dice:
            return 0.0
        counts = Counter(dice)
        unique_dice = set(dice)
        current_upper_score = sum(s for i, s in enumerate(state.rule_score) if i < 6 and s is not None)
        delta_63 = 63000 - current_upper_score
        U = sum(min(counts.get(f, 0) * f * 1000, delta_63) for f in range(1, 7)) / 1000
        P = sum(1 for c in counts.values() if c >= 2)
        T = sum(1 for c in counts.values() if c >= 3)
        Q = 1 if any(c >= 4 for c in counts.values()) else 0
        Y = 1 if any(c >= 5 for c in counts.values()) else 0
        small_straights = [{1, 2, 3, 4}, {2, 3, 4, 5}, {3, 4, 5, 6}]
        large_straights = [{1, 2, 3, 4, 5}, {2, 3, 4, 5, 6}]
        SS = sum(1 for s in small_straights if s.issubset(unique_dice))
        LS = sum(1 for s in large_straights if s.issubset(unique_dice))
        SS_prime = sum(1 for s in small_straights if len(s - unique_dice) == 1)
        LS_prime = sum(1 for s in large_straights if len(s - unique_dice) == 1)
        H = max(0, sum(dice) - 17.5)
        w = {
            'U': 800, 'P': 2000, 'T': 4500, 'Q': 9000, 'Y': 20000,
            'SS': 4000, 'LS': 9000, 'SS_prime': 1500, 'LS_prime': 2500, 'H': 600
        }
        available_rules = state.get_available_rules()
        if DiceRule.YACHT in available_rules: w['Y'] += 5000
        if DiceRule.LARGE_STRAIGHT in available_rules: w['LS'] += 3000
        if DiceRule.FOUR_OF_A_KIND in available_rules and DiceRule.FULL_HOUSE in available_rules: w['T'] += 1000
        num_upper_rules_left = sum(1 for r in available_rules if r.value < 6)
        if delta_63 >= 20000 and num_upper_rules_left > 3: w['U'] += 300
        option_value = (
            w['U'] * U + w['P'] * P + w['T'] * T + w['Q'] * Q + w['Y'] * Y +
            w['SS'] * SS + w['LS'] * LS + w['SS_prime'] * SS_prime +
            w['LS_prime'] * LS_prime + w['H'] * H
        )
        return option_value

    def _calculate_storage_value(self, dice: List[int], state: 'GameState') -> float:
        if not dice:
            return 0.0
        total_simulated_score = 0
        available_rules = state.get_available_rules()
        if not available_rules:
            return 0.0
        for _ in range(self.STORAGE_SIM_SAMPLES):
            new_dice = random.choices(range(1, 7), k=5)
            combined_hand = self.my_state.dice + dice + new_dice
            max_score_for_sim = 0
            for rule in available_rules:
                temp_put = DicePut(rule, combined_hand) 
                score = GameState.calculate_score(temp_put)
                if score > max_score_for_sim:
                    max_score_for_sim = score
            total_simulated_score += max_score_for_sim
        return total_simulated_score / self.STORAGE_SIM_SAMPLES

    # ================================ [전략의 핵심: 새로운 가치 평가 함수] ================================
    def _evaluate_dice_group_value(self, dice: List[int], state: 'GameState') -> float:
        """
        사용자의 전략(EV' = (P*S)*w)을 적용하여 덱의 가치를 동적으로 평가합니다.
        """
        # 1. 기대값(P*S)의 각 요소를 계산
        # S (Score): 즉시 얻을 수 있는 확정 점수
        # P*S (Potential): 앞으로 얻을 수 있는 잠재적 점수 (옵션 가치로 근사)
        immediate_score = self._calculate_immediate_score(dice, state)[0]
        option_value = self._calculate_option_value(dice, state)
        # S_storage: 시뮬레이션을 통한 미래 기대 점수
        storage_value = self._calculate_storage_value(dice, state)

        # 2. 동적 가중치(w)를 게임 상황에 맞게 설정
        # 기본 가중치 설정
        w_immediate = 1.0
        w_option = 0.7
        w_storage = 0.5
        
        # 게임 시점(라운드)에 따른 전략 변경
        if self.round <= 4:
            # 초반: 공격적 전략. 잠재력(옵션 가치)에 높은 가중치 부여
            w_option *= 1.2
            w_immediate *= 0.8
        elif self.round >= 9:
            # 후반: 안정적 전략. 확정 점수(즉시 점수)에 높은 가중치 부여
            w_immediate *= 1.3
            w_option *= 0.7
            w_storage *= 0.5

        # 점수 차이에 따른 전략 변경
        score_diff = state.get_total_score() - self.opp_state.get_total_score()
        if score_diff < -20000: # 크게 지고 있을 때
            # 공격성 증폭: 잠재력을 더욱 중요하게 생각하여 역전 노리기
            w_option *= 1.3
            w_storage *= 1.1
        elif score_diff > 20000: # 크게 이기고 있을 때
            # 안정성 증폭: 확정 점수를 우선하여 승리 굳히기
            w_immediate *= 1.2
            w_option *= 0.8

        # 3. 최종 가치 계산: EV' = (S * w_immediate) + (P*S * w_option) + (S_storage * w_storage)
        final_value = (immediate_score * w_immediate) + \
                      (option_value * w_option) + \
                      (storage_value * w_storage)
        
        return final_value
    # ===========================================================================================

    def calculate_bid(self, dice_a: List[int], dice_b: List[int]) -> Bid:
        self.round += 1
        self.seen_dice_counts.update(dice_a)
        self.seen_dice_counts.update(dice_b)
        
        # 1. 새로운 평가 함수를 통해 두 덱의 '전략적 가치'를 정교하게 계산
        value_a = self._evaluate_dice_group_value(dice_a, self.my_state)
        value_b = self._evaluate_dice_group_value(dice_b, self.my_state)
        my_choice_group = 'A' if value_a > value_b else 'B'
        value_difference = abs(value_a - value_b)

        # 2. 베팅액 블렌딩 및 동적 공격성 적용
        non_zero_bids = [b for b in self.opp_bid_history if b > 0]
        avg_opp_bid = (sum(non_zero_bids) / len(non_zero_bids)) if non_zero_bids else 500
        
        VALUE_TO_BID_RATIO = 20.0
        my_raw_bid = value_difference / VALUE_TO_BID_RATIO
        
        blended_bid = (my_raw_bid * 0.7) + (avg_opp_bid * 0.3)
        
        if self.round <= 4:
            aggression_multiplier = 0.9
        elif self.round <= 8:
            aggression_multiplier = 1.1
        else:
            aggression_multiplier = 1.3
            
        if self.contention_loss_streak >= 2:
            aggression_multiplier *= 1.2
        elif self.contention_win_streak >= 2:
            aggression_multiplier *= 0.9
        
        amount = blended_bid * aggression_multiplier

        # 3. 최종 베팅액 결정 (제약 없음)
        final_amount = max(1, int(amount))

        return Bid(my_choice_group, final_amount)

    def calculate_put(self) -> DicePut:
        my_hand = self.my_state.dice
        available_rules = self.my_state.get_available_rules()
        num_dice_to_choose = min(5, len(my_hand))

        if num_dice_to_choose <= 0:
            if available_rules: return DicePut(available_rules[0], [])
            return DicePut(DiceRule.CHOICE, [])

        dice_combos = list(itertools.combinations(my_hand, num_dice_to_choose))
        
        high_tier_rules = [DiceRule.FOUR_OF_A_KIND, DiceRule.FULL_HOUSE, DiceRule.LARGE_STRAIGHT]
        filled_high_tier_count = sum(1 for r in high_tier_rules if self.my_state.rule_score[r.value] is not None)
        yacht_filled = self.my_state.rule_score[DiceRule.YACHT.value] is not None

        focus_on_bonus = yacht_filled and filled_high_tier_count >= 3
        
        current_basic_score = sum(s for i, s in enumerate(self.my_state.rule_score) if i < 6 and s is not None)

        if focus_on_bonus:
            best_bonus_focus_put = None
            max_bonus_focus_value = -1
            
            number_rules_to_check = [r for r in available_rules if r.value <= 5]
            
            for rule in number_rules_to_check:
                for combo in dice_combos:
                    put = DicePut(rule, list(combo))
                    score = GameState.calculate_score(put)

                    if score == 0: continue

                    value = float(score)
                    if current_basic_score < 63000 and (current_basic_score + score) >= 63000:
                        value += 35000
                    
                    dice_num = rule.value + 1
                    seen_count = self.seen_dice_counts.get(dice_num, 0)
                    value += seen_count * 100

                    if value > max_bonus_focus_value:
                        max_bonus_focus_value = value
                        best_bonus_focus_put = put
            
            if best_bonus_focus_put:
                return best_bonus_focus_put
        
        priority_order = [
            DiceRule.YACHT, DiceRule.LARGE_STRAIGHT, DiceRule.FULL_HOUSE, DiceRule.FOUR_OF_A_KIND,
            DiceRule.SIX, DiceRule.FIVE, DiceRule.FOUR, DiceRule.SMALL_STRAIGHT,
            DiceRule.THREE, DiceRule.TWO, DiceRule.ONE, DiceRule.CHOICE,
        ]

        thresholds = {
            DiceRule.YACHT: 49000, DiceRule.LARGE_STRAIGHT: 29000, DiceRule.SMALL_STRAIGHT: 14000,
            DiceRule.FULL_HOUSE: 15000, DiceRule.FOUR_OF_A_KIND: 15000,
            DiceRule.SIX: 6000 * 4, DiceRule.FIVE: 5000 * 3, DiceRule.FOUR: 4000 * 3,
            DiceRule.THREE: 3000 * 3, DiceRule.TWO: 2000 * 3, DiceRule.ONE: 1000 * 3,
            DiceRule.CHOICE: 0,
        }
        
        six_rule_is_available = self.my_state.rule_score[DiceRule.SIX.value] is None

        for rule in priority_order:
            if rule in available_rules:
                best_value_for_rule = -1
                best_put_for_rule = None
                
                for combo in dice_combos:
                    dice_list = list(combo)
                    put = DicePut(rule, dice_list)
                    
                    if six_rule_is_available:
                        if (rule == DiceRule.LARGE_STRAIGHT or rule == DiceRule.FULL_HOUSE) and 6 in dice_list:
                            continue

                    score = GameState.calculate_score(put)
                    value = float(score)
                    is_basic_rule = rule.value <= 5
                    if is_basic_rule:
                        if current_basic_score < 63000 and (current_basic_score + score) >= 63000:
                            value += 35000
                        dice_num = rule.value + 1
                        seen_count = self.seen_dice_counts.get(dice_num, 0)
                        value += seen_count * 100
                    
                    if value > best_value_for_rule:
                        best_value_for_rule = value
                        best_put_for_rule = put
                
                if best_put_for_rule:
                    best_score = GameState.calculate_score(best_put_for_rule)
                    if best_score >= thresholds.get(rule, 1):
                        return best_put_for_rule

        discard_priority = [
            DiceRule.ONE, DiceRule.TWO, DiceRule.THREE, DiceRule.SMALL_STRAIGHT,
            DiceRule.FOUR, DiceRule.FIVE, DiceRule.FOUR_OF_A_KIND, DiceRule.FULL_HOUSE,
        ]

        for rule in discard_priority:
            if rule in available_rules:
                best_put_for_discard = None
                max_score_for_discard = -1
                lowest_sum_for_max_score = float('inf')
                
                for combo in dice_combos:
                    put = DicePut(rule, list(combo))
                    score = GameState.calculate_score(put)
                    combo_sum = sum(combo)

                    if score > max_score_for_discard:
                        max_score_for_discard = score
                        lowest_sum_for_max_score = combo_sum
                        best_put_for_discard = put
                    elif score == max_score_for_discard and score > -1:
                        if combo_sum < lowest_sum_for_max_score:
                            lowest_sum_for_max_score = combo_sum
                            best_put_for_discard = put
                
                if best_put_for_discard:
                    return best_put_for_discard

        if DiceRule.CHOICE in available_rules and dice_combos:
            best_choice_put = None
            max_choice_score = -1
            for combo in dice_combos:
                put = DicePut(DiceRule.CHOICE, list(combo))
                score = GameState.calculate_score(put)
                if score > max_choice_score:
                    max_choice_score = score
                    best_choice_put = put
            if best_choice_put:
                return best_choice_put

        if available_rules:
            dice_to_put = list(dice_combos[0]) if dice_combos else []
            return DicePut(available_rules[0], dice_to_put)
        else:
            dice_to_put = list(dice_combos[0]) if dice_combos else []
            return DicePut(DiceRule.CHOICE, dice_to_put)


    def update_get(self, dice_a: List[int], dice_b: List[int], my_bid: Bid, opp_bid: Bid, my_group: str):
        self.opp_bid_history.append(opp_bid.amount)
        is_contention = my_bid.group == opp_bid.group
        if is_contention:
            i_won = my_bid.group == my_group
            if i_won:
                self.contention_win_streak += 1
                self.contention_loss_streak = 0
            else:
                self.contention_loss_streak += 1
                self.contention_win_streak = 0
        if my_group == "A":
            self.my_state.add_dice(dice_a)
            self.opp_state.add_dice(dice_b)
        else:
            self.my_state.add_dice(dice_b)
            self.opp_state.add_dice(dice_a)
        my_bid_ok = my_bid.group == my_group
        self.my_state.bid(my_bid_ok, my_bid.amount)
        opp_group = "B" if my_group == "A" else "A"
        opp_bid_ok = opp_bid.group == opp_group
        self.opp_state.bid(opp_bid_ok, opp_bid.amount)

    def update_put(self, put: DicePut):
        self.my_state.use_dice(put)

    def update_set(self, put: DicePut):
        self.opp_state.use_dice(put)

class GameState:
    def __init__(self):
        self.dice = []
        self.rule_score: List[Optional[int]] = [None] * 12
        self.bid_score = 0

    def get_total_score(self) -> int:
        basic = sum(score for score in self.rule_score[0:6] if score is not None)
        bonus = 35000 if basic >= 63000 else 0
        combination = sum(score for score in self.rule_score[6:12] if score is not None)
        return basic + bonus + combination

    def get_available_rules(self) -> List[DiceRule]:
        return [DiceRule(i) for i, score in enumerate(self.rule_score) if score is None]

    def bid(self, is_successful: bool, amount: int):
        if is_successful:
            self.bid_score -= amount
        else:
            self.bid_score += amount

    def add_dice(self, new_dice: List[int]):
        self.dice.extend(new_dice)

    def use_dice(self, put: DicePut):
        assert put.rule is not None and self.rule_score[put.rule.value] is None, f"Rule {put.rule.name} already used"
        
        dice_to_put = put.dice
        for d in dice_to_put:
            if d in self.dice:
                self.dice.remove(d)
        
        self.rule_score[put.rule.value] = self.calculate_score(put)

    @staticmethod
    def calculate_score(put: DicePut) -> int:
        rule, dice = put.rule, put.dice
        if not dice:
            return 0

        counts = Counter(dice)
        if rule == DiceRule.ONE: return counts.get(1, 0) * 1000
        if rule == DiceRule.TWO: return counts.get(2, 0) * 2000
        if rule == DiceRule.THREE: return counts.get(3, 0) * 3000
        if rule == DiceRule.FOUR: return counts.get(4, 0) * 4000
        if rule == DiceRule.FIVE: return counts.get(5, 0) * 5000
        if rule == DiceRule.SIX: return counts.get(6, 0) * 6000
        if rule == DiceRule.CHOICE: return sum(dice) * 1000
        if rule == DiceRule.FOUR_OF_A_KIND:
            return sum(dice) * 1000 if any(c >= 4 for c in counts.values()) else 0
        
        if rule == DiceRule.FULL_HOUSE:
            count_values = sorted(counts.values(), reverse=True)
            is_full_house = False
            if len(count_values) >= 1 and count_values[0] >= 5:
                is_full_house = True
            elif len(count_values) >= 2 and count_values[0] >= 3 and count_values[1] >= 2:
                is_full_house = True
            return sum(dice) * 1000 if is_full_house else 0
            
        if rule == DiceRule.SMALL_STRAIGHT:
            unique_dice = set(dice)
            straights = [{1, 2, 3, 4}, {2, 3, 4, 5}, {3, 4, 5, 6}]
            return 15000 if any(s.issubset(unique_dice) for s in straights) else 0

        if rule == DiceRule.LARGE_STRAIGHT:
            unique_dice = set(dice)
            straights = [{1, 2, 3, 4, 5}, {2, 3, 4, 5, 6}]
            return 30000 if any(s.issubset(unique_dice) for s in straights) else 0
            
        if rule == DiceRule.YACHT:
            return 50000 if any(c >= 5 for c in counts.values()) else 0

        assert False, "Invalid rule"

def main():
    game = Game()
    # 게임 시작 시, 양쪽 플레이어의 bid_score를 100,000으로 설정해줍니다.
    # get_total_score 계산에 bid_score가 포함되지 않도록 GameState에서 관리
    # self.my_state.bid_score = 100000 -> bid_score는 0에서 시작하여 득실을 기록
    # self.opp_state.bid_score = 100000
    dice_a, dice_b = [0] * 5, [0] * 5
    my_bid = Bid("", 0)

    while True:
        try:
            line = input().strip()
            if not line: continue
            command, *args = line.split()
            if command == "READY":
                print("OK")
                continue
            if command == "ROLL":
                str_a, str_b = args
                dice_a = [int(c) for c in str_a]
                dice_b = [int(c) for c in str_b]
                my_bid = game.calculate_bid(dice_a, dice_b)
                print(f"BID {my_bid.group} {my_bid.amount}")
                continue
            if command == "GET":
                get_group, opp_group, opp_score = args
                opp_score = int(opp_score)
                opp_bid = Bid(opp_group, opp_score)
                game.update_get(dice_a, dice_b, my_bid, opp_bid, get_group)
                continue
            if command == "SCORE":
                put = game.calculate_put()
                assert put is not None, "calculate_put returned None"
                game.update_put(put)
                assert put.rule is not None
                dice_str = ''.join(map(str, sorted(put.dice))) if put.dice else ''
                print(f"PUT {put.rule.name} {dice_str}")
                continue
            if command == "SET":
                rule, str_dice = args
                dice = [int(c) for c in str_dice]
                game.update_set(DicePut(DiceRule[rule], dice))
                continue
            if command == "FINISH": break
            print(f"Invalid command: {command}", file=sys.stderr)
            sys.exit(1)
        except EOFError:
            break

if __name__ == "__main__":
    main()
