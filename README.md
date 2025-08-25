import random
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
        self.contention_win_streak = 0
        self.contention_loss_streak = 0

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
        
        my_potential_a, _ = self._evaluate_potential(self.my_state.dice + dice_a, self.my_state)
        my_potential_b, _ = self._evaluate_potential(self.my_state.dice + dice_b, self.my_state)
        
        potential_difference = abs(my_potential_a - my_potential_b)
        group_to_bid = 'A' if my_potential_a >= my_potential_b else 'B'
        
        non_zero_bids = [b for b in self.opp_bid_history if b > 0]
        avg_opp_bid = (sum(non_zero_bids) / len(non_zero_bids)) if non_zero_bids else 200

        base_bid = avg_opp_bid * 1.1
        value_component = potential_difference / 100
        
        amount = base_bid + value_component

        if self.contention_loss_streak >= 2:
            amount *= (1.0 + (self.contention_loss_streak * 0.2))
        elif self.contention_win_streak >= 2:
            amount *= 0.8
            
        amount += 500

        # [NEW] 후반 라운드 한정, 상위 족보 완성을 위한 과감한 베팅 로직
        if self.round >= 11:
            # 1. 아직 완성하지 못한 상위 족보 목록 확인
            late_game_rules_needed = {
                r for r in [
                    DiceRule.FOUR_OF_A_KIND, DiceRule.FULL_HOUSE,
                    DiceRule.SMALL_STRAIGHT, DiceRule.LARGE_STRAIGHT, DiceRule.YACHT
                ] if r in self.my_state.get_available_rules()
            }

            if late_game_rules_needed:
                # 2. 각 주사위 묶음이 필요한 상위 족보를 완성시킬 수 있는지 확인
                can_complete_with_a = any(
                    GameState.calculate_score(DicePut(rule, self.my_state.dice + dice_a)) > 0
                    for rule in late_game_rules_needed
                )
                can_complete_with_b = any(
                    GameState.calculate_score(DicePut(rule, self.my_state.dice + dice_b)) > 0
                    for rule in late_game_rules_needed
                )

                # 3. 내가 선택하려는 묶음이 역전의 발판이 될 수 있다면, 과감하게 베팅
                if (group_to_bid == 'A' and can_complete_with_a) or \
                   (group_to_bid == 'B' and can_complete_with_b):
                    amount += 10000

        return Bid(group_to_bid, max(1, int(amount)))

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
            DiceRule.YACHT,
            DiceRule.LARGE_STRAIGHT,
            DiceRule.FULL_HOUSE,
            DiceRule.FOUR_OF_A_KIND,
            DiceRule.SIX,
            DiceRule.FIVE,
            DiceRule.FOUR,
            DiceRule.SMALL_STRAIGHT,
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
                best_value_for_rule = -1
                best_put_for_rule = None
                
                for combo in dice_combos:
                    dice_list = list(combo)
                    put = DicePut(rule, dice_list)
                    
                    if rule == DiceRule.YACHT:
                        if 5 in dice_list or 6 in dice_list:
                            continue

                    if rule == DiceRule.CHOICE:
                        if 6 in dice_list:
                            continue
                    
                    if rule == DiceRule.FOUR_OF_A_KIND:
                        if 6 in dice_list:
                            continue
                    
                    if six_rule_is_available:
                        if (rule == DiceRule.LARGE_STRAIGHT or rule == DiceRule.FULL_HOUSE) and 6 in dice_list:
                            continue

                    score = GameState.calculate_score(put)
                    
                    value = float(score)
                    
                    is_low_basic_rule = rule.value <= 3
                    if is_low_basic_rule:
                        if 6 in dice_list:
                            continue
                        if 5 in dice_list:
                            value -= 5000
                    
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
            DiceRule.ONE,
            DiceRule.TWO,
            DiceRule.THREE,
            DiceRule.SMALL_STRAIGHT,
            DiceRule.FOUR,
            DiceRule.FOUR_OF_A_KIND,
            DiceRule.FIVE,
            DiceRule.FULL_HOUSE,
        ]

        for rule in discard_priority:
            if rule in available_rules:
                best_put_for_discard = None
                max_score_for_discard = -1
                
                for combo in dice_combos:
                    put = DicePut(rule, list(combo))
                    score = GameState.calculate_score(put)

                    if score > max_score_for_discard:
                        max_score_for_discard = score
                        best_put_for_discard = put
                
                if best_put_for_discard:
                    return best_put_for_discard

        if DiceRule.CHOICE in available_rules and dice_combos:
            best_choice_put = None
            max_choice_score = -1
            for combo in dice_combos:
                if 6 in list(combo):
                    continue
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


    # ============================== [필수 구현 끝] ==============================

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
    game.my_state.bid_score = 100000
    game.opp_state.bid_score = 100000
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
                opp_bid = Bid(opp_group, opp_score)
                game.update_get(
                    dice_a, dice_b, my_bid, opp_bid, get_group
                )
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

            if command == "FINISH":
                break

            print(f"Invalid command: {command}", file=sys.stderr)
            sys.exit(1)

        except EOFError:
            break


if __name__ == "__main__":
    main()
