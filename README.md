import sys
import copy

class Game:
    """
    상대방의 반격까지 예측하고, 가장자리 선호 전략을 포함한 AI 게임 클래스.
    미니맥스(Minimax) 기본 원리를 적용하여 한 수 앞을 내다봅니다.
    """
    def __init__(self, board, first):
        self.board = board
        self.first = first
        self.passed = False
        
        self.rows = len(board)
        self.cols = len(board[0])
        
        # 점령 상태 배열 초기화
        self.my_area = [[False for _ in range(self.cols)] for _ in range(self.rows)]
        self.opp_area = [[False for _ in range(self.cols)] for _ in range(self.rows)]
        
        # 누적 합 배열 초기화
        self.prefix_sum = [[0 for _ in range(self.cols + 1)] for _ in range(self.rows + 1)]
        self._rebuild_prefix_sum()

    def _rebuild_prefix_sum(self):
        """보드 변경 후 prefix_sum을 최신 상태로 갱신합니다."""
        for r in range(self.rows):
            for c in range(self.cols):
                self.prefix_sum[r+1][c+1] = (
                    self.prefix_sum[r][c+1] +
                    self.prefix_sum[r+1][c] -
                    self.prefix_sum[r][c] +
                    self.board[r][c]
                )

    def isValid(self, r1, c1, r2, c2):
        """주어진 직사각형이 유효한 수인지 검사합니다 (합 10, 경계 조건)."""
        # 좌표 범위 유효성 검사 추가
        if not (0 <= r1 < self.rows and 0 <= c1 < self.cols and \
                r1 <= r2 < self.rows and c1 <= c2 < self.cols):
            return False

        sums = self.prefix_sum[r2 + 1][c2 + 1] \
             - self.prefix_sum[r1][c2 + 1] \
             - self.prefix_sum[r2 + 1][c1] \
             + self.prefix_sum[r1][c1]

        if sums != 10:
            return False

        # 숫자가 직사각형의 모든 경계에 닿아 있는지 확인
        r1fit = c1fit = r2fit = c2fit = False
        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                if self.board[r][c] != 0:
                    if r == r1: r1fit = True
                    if r == r2: r2fit = True
                    if c == c1: c1fit = True
                    if c == c2: c2fit = True
        
        return r1fit and r2fit and c1fit and c2fit

    def calculateMove(self, _myTime, _oppTime):
        """
        미니맥스, 방어, 가장자리 선호 전략을 통합하여 최적의 수를 계산합니다.
        """
        best_move_for_me = (-1, -1, -1, -1)
        max_lookahead_score = -float('inf') 

        my_possible_moves = self._find_all_valid_moves()

        if not my_possible_moves:
            return (-1, -1, -1, -1)

        for my_move in my_possible_moves:
            # 가상 게임을 만들어 내가 수를 뒀을 때의 상황을 시뮬레이션
            virtual_game = copy.deepcopy(self)
            virtual_game.updateMove(*my_move, isMyMove=True)
            
            # 가상 게임에서 상대방의 최선의 반격을 찾음
            opponent_best_reply = virtual_game._find_best_opponent_move()
            
            # 상대방이 반격한 후의 상태를 시뮬레이션
            if opponent_best_reply != (-1, -1, -1, -1):
                virtual_game.updateMove(*opponent_best_reply, isMyMove=False)
            
            # 시뮬레이션 종료 후의 최종 보드 점수를 계산
            current_lookahead_score = virtual_game._evaluate_board_state()
            
            # --- [조건 5] 가장자리 선호 전략 적용 ---
            # 최종 평가 점수에 가장자리 보너스 점수를 더해줍니다.
            edge_bonus = self._get_edge_bonus(my_move)
            current_lookahead_score += edge_bonus
            
            if current_lookahead_score > max_lookahead_score:
                max_lookahead_score = current_lookahead_score
                best_move_for_me = my_move
                
        return best_move_for_me

    def _find_all_valid_moves(self):
        """현재 보드에서 가능한 모든 유효한 수를 찾아 리스트로 반환합니다."""
        moves = []
        for r1 in range(self.rows):
            for c1 in range(self.cols):
                for r2 in range(r1, self.rows):
                    for c2 in range(c1, self.cols):
                        if self.isValid(r1, c1, r2, c2):
                            moves.append((r1, c1, r2, c2))
        return moves

    def _find_best_opponent_move(self):
        """
        상대방 입장에서 가장 좋은 단기적인 수를 찾습니다.
        이 과정 자체가 '나의 수'에 대한 '상대방의 최선 반격'을 찾는 것이므로,
        방어적 플레이의 핵심 로직이 됩니다.
        """
        best_move = (-1, -1, -1, -1)
        max_score = -1
        
        possible_moves = self._find_all_valid_moves()
        for move in possible_moves:
            # --- [조건 4] 방어적 플레이 강화 ---
            # 상대방의 입장에서 점수를 계산하여(isMyMove=False),
            # 나의 땅을 얼마나 뺏을 수 있는지를 정확하게 평가합니다.
            score = self.calculate_hybrid_score(*move, isMyMove=False)
            if score > max_score:
                max_score = score
                best_move = move
        return best_move

    def calculate_hybrid_score(self, r1, c1, r2, c2, isMyMove):
        """
        단기적인 수의 가치를 평가합니다. isMyMove 플래그로 누구의 관점인지 명시합니다.
        """
        stolen_cells = 0
        claimed_cells = 0
        
        # isMyMove 값에 따라 나의 영역과 상대 영역을 설정
        my_current_area = self.my_area if isMyMove else self.opp_area
        opp_current_area = self.opp_area if isMyMove else self.my_area
        
        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                if opp_current_area[r][c]: # 현재 플레이어의 상대방 땅이라면
                    stolen_cells += 1
                elif not my_current_area[r][c]: # 아직 현재 플레이어의 땅이 아니라면
                    claimed_cells += 1
        
        return (stolen_cells * 10) + claimed_cells

    def _evaluate_board_state(self):
        """(내 전체 영역 - 상대 전체 영역)으로 보드의 최종 유불리를 평가합니다."""
        my_total_area = sum(row.count(True) for row in self.my_area)
        opp_total_area = sum(row.count(True) for row in self.opp_area)
        return my_total_area - opp_total_area

    def _get_edge_bonus(self, move):
        """
        [조건 5] 수가 보드의 가장자리에 닿아있으면 작은 보너스 점수를 반환합니다.
        """
        r1, c1, r2, c2 = move
        if r1 == 0 or r2 == self.rows - 1 or c1 == 0 or c2 == self.cols - 1:
            return 0.1 # 다른 점수에 영향을 미치지 않는 작은 값
        return 0

    def updateOpponentAction(self, action, _time):
        """상대방의 수를 게임 상태에 반영합니다."""
        self.updateMove(*action, isMyMove=False)

    def updateMove(self, r1, c1, r2, c2, isMyMove):
        """게임의 보드와 영역 상태를 업데이트합니다."""
        if r1 == c1 == r2 == c2 == -1:
            self.passed = True
            return

        area_to_update = self.my_area if isMyMove else self.opp_area
        opp_area_to_clear = self.opp_area if isMyMove else self.my_area

        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                self.board[r][c] = 0
                area_to_update[r][c] = True
                opp_area_to_clear[r][c] = False

        self._rebuild_prefix_sum()
        self.passed = False


def main():
    """게임 서버와의 통신을 처리하는 메인 루프입니다."""
    game = None
    first = False # first 변수 초기화

    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue

        parts = line.split()
        command, *param = parts

        if command == "READY":
            turn = param[0]
            first = (turn == "FIRST")
            print("OK", flush=True)
            continue

        if command == "INIT":
            board = [list(map(int, row)) for row in param]
            game = Game(board, first)
            continue

        if command == "TIME":
            myTime, oppTime = map(int, param)
            ret = game.calculateMove(myTime, oppTime)
            game.updateMove(*ret, isMyMove=True)
            print(*ret, flush=True)
            continue

        if command == "OPP":
            r1, c1, r2, c2, time = map(int, param)
            game.updateOpponentAction((r1, c1, r2, c2), time)
            continue

        if command == "FINISH":
            break


if __name__ == "__main__":
    main()
