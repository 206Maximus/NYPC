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
        self.opp_bid_history = [] # 상대방의 입찰 기록

    # ================================ [필수 구현] ================================
    def _evaluate_potential(self, dice: List[int]) -> Tuple[int, Optional[DiceRule]]:
        """
        주어진 5개의 주사위로 얻을 수 있는 최대 잠재 점수와 그 때의 규칙을 계산합니다.
        아직 사용하지 않은 규칙만을 대상으로 합니다.
        """
        best_score = -1
        best_rule = None
        available_rules = [
            DiceRule(i)
            for i, score in enumerate(self.my_state.rule_score)
            if score is None
        ]

        for rule in available_rules:
            score = GameState.calculate_score(DicePut(rule, dice))
            if score > best_score:
                best_score = score
                best_rule = rule

        return best_score, best_rule

    def calculate_bid(self, dice_a: List[int], dice_b: List[int]) -> Bid:
        """
        상대방의 평균 입찰액을 기준으로, 현재 게임 상황과 주사위 가치를
        고려하여 입찰 전략을 결정합니다. 총점의 20%를 넘지 않도록 제한합니다.
        """
        self.round += 1

        # 1. 주사위 가치 평가
        potential_a, _ = self._evaluate_potential(dice_a)
        potential_b, _ = self._evaluate_potential(dice_b)

        if potential_a > potential_b:
            group = "A"
            chosen_potential = potential_a
            other_potential = potential_b
        else:
            if potential_a == potential_b and sum(dice_a) > sum(dice_b):
                 group = "A"
                 chosen_potential = potential_a
                 other_potential = potential_b
            else:
                group = "B"
                chosen_potential = potential_b
                other_potential = potential_a

        # 2. 상대방 평균 입찰액 계산
        if not self.opp_bid_history:
            opp_avg_bid = 1  # 기록이 없으면 보수적인 초기값 설정
        else:
            opp_avg_bid = (sum(self.opp_bid_history) / len(self.opp_bid_history) ) + 1
            
        # 3. 입찰액 계산 (상대 평균 기반)
        # 기본 입찰액: 상대의 평균 입찰액
        amount = opp_avg_bid

        # 조정 1: 내가 원하는 묶음의 상대적 가치 반영
        amount += (chosen_potential - other_potential) / 3.0

        # 조정 2: 현재 점수 상황 반영 (승/패)
        score_diff = self.my_state.get_total_score() - self.opp_state.get_total_score()
        if score_diff < 0: # 지고 있을 때
            amount += abs(score_diff) / 5.0
        else: # 이기고 있을 때
            amount -= score_diff / 6.0

        # 4. 최대 입찰액 제한 (리스크 관리)
        my_total_score = self.my_state.get_total_score()
        # 총점이 0 이하일 경우를 대비해 최소한의 입찰액 한도는 보장
        max_bid_from_score = max(1, my_total_score * 0.10)
        
        # 최종 입찰액 결정
        # 음수 베팅은 불가능
        final_amount = max(0, int(amount))
        # 내 총점의 20%와 게임 최대 한도(100000) 중 작은 값으로 제한
        final_amount = min(final_amount, int(max_bid_from_score), 100000)

        return Bid(group, final_amount)


    def calculate_put(self) -> DicePut:
        """
        현재 보유한 주사위와 사용 가능한 규칙을 모두 조합하여
        최고의 점수를 낼 수 있는 조합을 찾아 반환합니다.
        """
        best_put = None
        best_score = -1

        available_rules = [
            DiceRule(i)
            for i, score in enumerate(self.my_state.rule_score)
            if score is None
        ]

        # 보유한 주사위가 5개보다 많으면 5개를 선택하는 모든 조합을 고려
        num_dice_to_choose = 5
        if len(self.my_state.dice) < num_dice_to_choose:
            num_dice_to_choose = len(self.my_state.dice)
            
        # 중복된 조합을 피하기 위해 set 사용
        if num_dice_to_choose > 0:
            dice_combos = set(itertools.combinations(self.my_state.dice, num_dice_to_choose))
        else:
            dice_combos = []

        if not dice_combos and num_dice_to_choose > 0: # 5개 미만일 경우
            dice_combos.add(tuple(self.my_state.dice))

        for combo in dice_combos:
            dice_list = list(combo)
            for rule in available_rules:
                current_score = GameState.calculate_score(DicePut(rule, dice_list))
                
                if current_score > best_score:
                    best_score = current_score
                    best_put = DicePut(rule, dice_list)

        # 만약 어떤 규칙으로도 점수를 낼 수 없다면(best_score <= 0),
        # 가장 점수 기대값이 낮은 규칙부터 사용
        if best_score <= 0:
            low_priority_rules = [DiceRule.ONE, DiceRule.TWO, DiceRule.THREE, DiceRule.CHOICE]
            dice_to_put = self.my_state.dice[:num_dice_to_choose] if num_dice_to_choose > 0 else []

            for rule in low_priority_rules:
                if self.my_state.rule_score[rule.value] is None:
                    best_put = DicePut(rule, dice_to_put)
                    return best_put
            
            if best_put is None and available_rules:
                 first_available_rule = available_rules[0]
                 best_put = DicePut(first_available_rule, dice_to_put)


        return best_put

    # ============================== [필수 구현 끝] ==============================

    def update_get(
        self,
        dice_a: List[int],
        dice_b: List[int],
        my_bid: Bid,
        opp_bid: Bid,
        my_group: str,
    ):
        """입찰 결과를 받아서 상태 업데이트"""
        # 상대방의 입찰 기록 저장
        self.opp_bid_history.append(opp_bid.amount)

        # 그룹에 따라 주사위 분배
        if my_group == "A":
            self.my_state.add_dice(dice_a)
            self.opp_state.add_dice(dice_b)
        else:
            self.my_state.add_dice(dice_b)
            self.opp_state.add_dice(dice_a)

        # 입찰 결과에 따른 점수 반영
        my_bid_ok = my_bid.group == my_group
        self.my_state.bid(my_bid_ok, my_bid.amount)

        opp_group = "B" if my_group == "A" else "A"
        opp_bid_ok = opp_bid.group == opp_group
        self.opp_state.bid(opp_bid_ok, opp_bid.amount)

    def update_put(self, put: DicePut):
        """내가 주사위를 배치한 결과 반영"""
        self.my_state.use_dice(put)

    def update_set(self, put: DicePut):
        """상대가 주사위를 배치한 결과 반영"""
        self.opp_state.use_dice(put)


# 팀의 현재 상태를 관리하는 클래스
class GameState:
    def __init__(self):
        self.dice = []  # 현재 보유한 주사위 목록
        self.rule_score: List[Optional[int]] = [
            None
        ] * 12  # 각 규칙별 획득 점수 (사용하지 않았다면 None)
        self.bid_score = 0  # 입찰로 얻거나 잃은 총 점수

    def get_total_score(self) -> int:
        """현재까지 획득한 총 점수 계산 (상단/하단 점수 + 보너스 + 입찰 점수)"""
        basic = bonus = combination = 0

        # 기본 점수 규칙 계산 (ONE ~ SIX)
        basic = sum(score for score in self.rule_score[0:6] if score is not None)
        bonus = 35000 if basic >= 63000 else 0
        combination = sum(score for score in self.rule_score[6:12] if score is not None)

        return basic + bonus + combination + self.bid_score

    def bid(self, is_successful: bool, amount: int):
        """입찰 결과에 따른 점수 반영"""
        if is_successful:
            self.bid_score -= amount  # 성공시 베팅 금액만큼 점수 차감
        else:
            self.bid_score += amount  # 실패시 베팅 금액만큼 점수 획득

    def add_dice(self, new_dice: List[int]):
        """새로운 주사위들을 보유 목록에 추가"""
        self.dice.extend(new_dice)

    def use_dice(self, put: DicePut):
        """주사위를 사용하여 특정 규칙에 배치"""
        # 이미 사용한 규칙인지 확인
        assert (
            put.rule is not None and self.rule_score[put.rule.value] is None
        ), "Rule already used"

        for d in put.dice:
            # 주사위 목록에 있는 주사위 제거
            if d in self.dice:
                self.dice.remove(d)

        # 해당 규칙의 점수 계산 및 저장
        assert put.rule is not None
        self.rule_score[put.rule.value] = self.calculate_score(put)

    @staticmethod
    def calculate_score(put: DicePut) -> int:
        """규칙에 따른 점수를 계산하는 함수"""
        rule, dice = put.rule, put.dice
        
        if not dice:
            return 0

        # 기본 규칙 점수 계산 (해당 숫자에 적힌 수의 합 × 1000점)
        if rule == DiceRule.ONE:
            return sum(d for d in dice if d == 1) * 1000
        if rule == DiceRule.TWO:
            return sum(d for d in dice if d == 2) * 1000
        if rule == DiceRule.THREE:
            return sum(d for d in dice if d == 3) * 1000
        if rule == DiceRule.FOUR:
            return sum(d for d in dice if d == 4) * 1000
        if rule == DiceRule.FIVE:
            return sum(d for d in dice if d == 5) * 1000
        if rule == DiceRule.SIX:
            return sum(d for d in dice if d == 6) * 1000
        if rule == DiceRule.CHOICE:  # 주사위에 적힌 모든 수의 합 × 1000점
            return sum(dice) * 1000
        if (
            rule == DiceRule.FOUR_OF_A_KIND
        ):  # 같은 수가 적힌 주사위가 4개 있다면, 주사위에 적힌 모든 수의 합 × 1000점, 아니면 0
            counts = Counter(dice)
            ok = any(count >= 4 for count in counts.values())
            return sum(dice) * 1000 if ok else 0
        if (
            rule == DiceRule.FULL_HOUSE
        ):  # 3개의 주사위에 적힌 수가 서로 같고, 다른 2개의 주사위에 적힌 수도 서로 같으면 주사위에 적힌 모든 수의 합 × 1000점, 아닐 경우 0점
            counts = Counter(dice).values()
            # Yacht(5개 동일)도 Full House로 인정
            ok = sorted(counts) == [2, 3] or 5 in counts
            return sum(dice) * 1000 if ok else 0
        if (
            rule == DiceRule.SMALL_STRAIGHT
        ):  # 4개의 주사위에 적힌 수가 1234, 2345, 3456중 하나로 연속되어 있을 때, 15000점, 아닐 경우 0점
            unique_dice = sorted(list(set(dice)))
            straights = [{1, 2, 3, 4}, {2, 3, 4, 5}, {3, 4, 5, 6}]
            ok = any(s.issubset(set(unique_dice)) for s in straights)
            return 15000 if ok else 0
        if (
            rule == DiceRule.LARGE_STRAIGHT
        ):  # 5개의 주사위에 적힌 수가 12345, 23456중 하나로 연속되어 있을 때, 30000점, 아닐 경우 0점
            unique_dice_set = set(dice)
            ok = (
                unique_dice_set == {1, 2, 3, 4, 5}
                or unique_dice_set == {2, 3, 4, 5, 6}
            )
            return 30000 if ok else 0
        if (
            rule == DiceRule.YACHT
        ):  # 5개의 주사위에 적힌 수가 모두 같을 때 50000점, 아닐 경우 0점
            ok = len(set(dice)) == 1 and len(dice) == 5
            return 50000 if ok else 0

        assert False, "Invalid rule"


def main():
    game = Game()

    # 입찰 라운드에서 나온 주사위들
    dice_a, dice_b = [0] * 5, [0] * 5
    # 내가 마지막으로 한 입찰 정보
    my_bid = Bid("", 0)

    while True:
        try:
            line = input().strip()
            if not line:
                continue

            command, *args = line.split()

            if command == "READY":
                # 게임 시작
                print("OK")
                continue

            if command == "ROLL":
                # 주사위 굴리기 결과 받기
                str_a, str_b = args
                for i, c in enumerate(str_a):
                    dice_a[i] = int(c)  # 문자를 숫자로 변환
                for i, c in enumerate(str_b):
                    dice_b[i] = int(c)  # 문자를 숫자로 변환
                my_bid = game.calculate_bid(dice_a, dice_b)
                print(f"BID {my_bid.group} {my_bid.amount}")
                continue

            if command == "GET":
                # 주사위 받기
                get_group, opp_group, opp_score = args
                opp_score = int(opp_score)
                game.update_get(
                    dice_a, dice_b, my_bid, Bid(opp_group, opp_score), get_group
                )
                continue

            if command == "SCORE":
                # 주사위 골라서 배치하기
                put = game.calculate_put()
                game.update_put(put)
                assert put.rule is not None
                print(f"PUT {put.rule.name} {''.join(map(str, sorted(put.dice)))}")
                continue

            if command == "SET":
                # 상대의 주사위 배치
                rule, str_dice = args
                dice = [int(c) for c in str_dice]
                game.update_set(DicePut(DiceRule[rule], dice))
                continue

            if command == "FINISH":
                # 게임 종료
                break

            # 알 수 없는 명령어 처리
            print(f"Invalid command: {command}", file=sys.stderr)
            sys.exit(1)

        except EOFError:
            break


if __name__ == "__main__":
    main()
