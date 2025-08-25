from enum import Enum
from dataclasses import dataclass
from typing import List, Optional, Tuple
import sys
import itertools
from collections import Counter


# 가능한 주사위 규칙들을 나타내는 enum
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


# 입찰 방법을 나타내는 데이터클래스
@dataclass
class Bid:
    group: str  # 입찰 그룹 ('A' 또는 'B')
    amount: int  # 입찰 금액


# 주사위 배치 방법을 나타내는 데이터클래스
@dataclass
class DicePut:
    rule: DiceRule  # 배치 규칙
    dice: List[int]  # 배치할 주사위 목록


# 게임 상태를 관리하는 클래스
class Game:
    def __init__(self):
        self.my_state = GameState()  # 내 팀의 현재 상태
        self.opp_state = GameState()  # 상대 팀의 현재 상태
        self.round = 0
        self.opp_bid_history = []  # 상대방의 입찰 기록
        self.seen_dice_counts = Counter()  # 게임 전체에 등장한 주사위 숫자 카운트

    # ================================ [필수 구현] ================================
    def _evaluate_potential(self, dice: List[int], state: 'GameState') -> Tuple[int, Optional[DiceRule]]:
        best_score = -1
        best_rule = None
        available_rules = [
            DiceRule(i)
            for i, score in enumerate(state.rule_score)
            if score is None
        ]

        for rule in available_rules:
            score = GameState.calculate_score(DicePut(rule, dice))
            if score > best_score:
                best_score = score
                best_rule = rule

        return (best_score, best_rule) if best_score > -1 else (0, None)

    def calculate_bid(self, dice_a: List[int], dice_b: List[int]) -> Bid:
        self.round += 1
        self.seen_dice_counts.update(dice_a)
        self.seen_dice_counts.update(dice_b)
        
        if self.round == 1:
            my_potential_a, _ = self._evaluate_potential(dice_a, self.my_state)
            my_potential_b, _ = self._evaluate_potential(dice_b, self.my_state)
            group_to_bid = 'A' if my_potential_a >= my_potential_b else 'B'
            return Bid(group_to_bid, 1)

        non_zero_bids = [b for b in self.opp_bid_history if b > 0]
        avg_opp_bid = (sum(non_zero_bids) / len(non_zero_bids)) if non_zero_bids else 100

        base_bid = max(1, int(avg_opp_bid))
        
        my_potential_a, my_rule_a = self._evaluate_potential(dice_a, self.my_state)
        my_potential_b, my_rule_b = self._evaluate_potential(dice_b, self.my_state)
        
        group_to_bid = 'A' if my_potential_a >= my_potential_b else 'B'
        
        # === 중/후반 통합 공격적 입찰 로직 (1.9배) ===
        critical_bid = max(1, int(avg_opp_bid * 1.9))
        
        PRIORITY_RULES = {
            DiceRule.YACHT, DiceRule.LARGE_STRAIGHT, DiceRule.SMALL_STRAIGHT, 
            DiceRule.SIX, DiceRule.FIVE, DiceRule.FULL_HOUSE, DiceRule.FOUR_OF_A_KIND
        }
        a_is_prio = my_rule_a in PRIORITY_RULES and self.my_state.rule_score[my_rule_a.value] is None
        b_is_prio = my_rule_b in PRIORITY_RULES and self.my_state.rule_score[my_rule_b.value] is None

        if a_is_prio or b_is_prio:
            if a_is_prio and not b_is_prio:
                return Bid('A', critical_bid)
            if not a_is_prio and b_is_prio:
                return Bid('B', critical_bid)
            
            return Bid(group_to_bid, critical_bid)
        else:
            return Bid(group_to_bid, base_bid)

    def calculate_put(self) -> DicePut:
        my_hand = self.my_state.dice
        available_rules = self.my_state.get_available_rules()
        num_dice_to_choose = min(5, len(my_hand))

        if num_dice_to_choose <= 0:
            if available_rules: return DicePut(available_rules[0], [])
            return DicePut(DiceRule.CHOICE, [])

        dice_combos = list(itertools.combinations(my_hand, num_dice_to_choose))
        
        priority_order = [
            DiceRule.SIX,
            DiceRule.YACHT,
            DiceRule.LARGE_STRAIGHT,
            DiceRule.FULL_HOUSE,
            DiceRule.FOUR_OF_A_KIND,
            DiceRule.SMALL_STRAIGHT,
            DiceRule.FIVE,
            DiceRule.FOUR,
            DiceRule.THREE,
            DiceRule.TWO,
            DiceRule.ONE,
            DiceRule.CHOICE,
        ]

        thresholds = {
            DiceRule.YACHT: 49000,
            DiceRule.LARGE_STRAIGHT: 29000,
            DiceRule.SMALL_STRAIGHT: 14000,
            DiceRule.FULL_HOUSE: 15000,
            DiceRule.FOUR_OF_A_KIND: 15000,
            DiceRule.SIX: 6000 * 4,
            DiceRule.FIVE: 5000 * 3,
            DiceRule.FOUR: 4000 * 3,
            DiceRule.THREE: 3000 * 3,
            DiceRule.TWO: 2000 * 3,
            DiceRule.ONE: 1000 * 3,
            DiceRule.CHOICE: 0,
        }
        
        six_rule_is_available = self.my_state.rule_score[DiceRule.SIX.value] is None

        for rule in priority_order:
            if rule in available_rules:
                best_score_for_rule = -1
                best_dice_for_rule = []
                
                for combo in dice_combos:
                    dice_list = list(combo)
                    
                    if six_rule_is_available:
                        rules_to_save_six_from = {
                            DiceRule.ONE, DiceRule.TWO, DiceRule.THREE, DiceRule.FOUR, DiceRule.FIVE,
                            DiceRule.FULL_HOUSE, DiceRule.LARGE_STRAIGHT
                        }
                        if rule in rules_to_save_six_from and 6 in dice_list:
                            continue
                        
                        if rule == DiceRule.CHOICE and dice_list.count(6) > 1:
                            continue
                        if rule == DiceRule.FOUR_OF_A_KIND and dice_list.count(6) > 1:
                            continue

                    score = GameState.calculate_score(DicePut(rule, dice_list))
                    if score > best_score_for_rule:
                        best_score_for_rule = score
                        best_dice_for_rule = dice_list
                
                if best_score_for_rule >= thresholds.get(rule, 1):
                    return DicePut(rule, best_dice_for_rule)

        best_fallback_put = None
        best_fallback_score = -1
        for rule in available_rules:
            for combo in dice_combos:
                dice_list = list(combo)

                if six_rule_is_available:
                    rules_to_save_six_from = {
                        DiceRule.ONE, DiceRule.TWO, DiceRule.THREE, DiceRule.FOUR, DiceRule.FIVE,
                        DiceRule.FULL_HOUSE, DiceRule.LARGE_STRAIGHT
                    }
                    if rule in rules_to_save_six_from and 6 in dice_list:
                        continue
                    if rule == DiceRule.CHOICE and dice_list.count(6) > 1:
                        continue
                    if rule == DiceRule.FOUR_OF_A_KIND and dice_list.count(6) > 1:
                        continue
                
                score = GameState.calculate_score(DicePut(rule, dice_list))
                if score > best_fallback_score:
                    best_fallback_score = score
                    best_fallback_put = DicePut(rule, dice_list)

        if best_fallback_put:
            return best_fallback_put

        low_value_rules = [DiceRule.ONE, DiceRule.TWO, DiceRule.CHOICE]
        if dice_combos:
            for rule in low_value_rules:
                if rule in available_rules:
                    return DicePut(rule, list(dice_combos[0]))
        
        if available_rules:
            return DicePut(available_rules[0], list(dice_combos[0]) if dice_combos else [])
        
        return DicePut(DiceRule.CHOICE, [])

    # ============================== [필수 구현 끝] ==============================

    def update_get(self, dice_a: List[int], dice_b: List[int], my_bid: Bid, opp_bid: Bid, my_group: str):
        self.opp_bid_history.append(opp_bid.amount)

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
        return basic + bonus + combination + self.bid_score

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
        for d in put.dice:
            if d in self.dice:
                self.dice.remove(d)
        assert put.rule is not None
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
            return sum(dice) * 1000 if sorted(counts.values()) in ([2, 3], [5]) else 0
        if rule == DiceRule.SMALL_STRAIGHT:
            unique_dice = set(dice)
            straights = [{1, 2, 3, 4}, {2, 3, 4, 5}, {3, 4, 5, 6}]
            return 15000 if any(s.issubset(unique_dice) for s in straights) else 0
        if rule == DiceRule.LARGE_STRAIGHT:
            unique_dice = set(dice)
            return 30000 if unique_dice in [{1, 2, 3, 4, 5}, {2, 3, 4, 5, 6}] else 0
        if rule == DiceRule.YACHT:
            return 50000 if 5 in counts.values() else 0

        assert False, "Invalid rule"

def main():
    game = Game()
    dice_a, dice_b = [0] * 5, [0] * 5
    my_bid = Bid("", 0)

    while True:
        try:
            line = input().strip()
            if not line:
                continue

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
                game.update_get(
                    dice_a, dice_b, my_bid, Bid(opp_group, opp_score), get_group
                )
                continue

            if command == "SCORE":
                put = game.calculate_put()
                assert put is not None, "calculate_put returned None"
                game.update_put(put)
                assert put.rule is not None
                print(f"PUT {put.rule.name} {''.join(map(str, sorted(put.dice)))}")
                continue

            if command == "SET":
                rule, str_dice = args
                dice = [int(c) for c in str_dice]
                game.update_set(DicePut(DiceRule[rule], dice))
                continue

            if command == "FINISH":
                break

            print(f"Invalid command: {command}", file=sys.stderr)
            sys.exit(1)

        except EOFError:
            break


if __name__ == "__main__":
    main()
