import sys
import copy

class Game:
    """
    [전략 수정] 위치 플레이를 대폭 강화하고, 영토 침략에 대한 강력한 페널티 시스템을 도입한 AI.
    """
    def __init__(self, board, first):
        self.board = board
        self.first = first
        self.passed = False
        
        self.rows = len(board)
        self.cols = len(board[0])
        
        self.my_area = [[False for _ in range(self.cols)] for _ in range(self.rows)]
        self.opp_area = [[False for _ in range(self.cols)] for _ in range(self.rows)]
        
        self.prefix_sum = [[0 for _ in range(self.cols + 1)] for _ in range(self.rows + 1)]
        self._rebuild_prefix_sum()

    def _rebuild_prefix_sum(self):
        for r in range(self.rows):
            for c in range(self.cols):
                self.prefix_sum[r+1][c+1] = (
                    self.prefix_sum[r][c+1] +
                    self.prefix_sum[r+1][c] -
                    self.prefix_sum[r][c] +
                    self.board[r][c]
                )

    def isValid(self, r1, c1, r2, c2):
        if not (0 <= r1 < self.rows and 0 <= c1 < self.cols and \
                r1 <= r2 < self.rows and c1 <= c2 < self.cols):
            return False

        sums = self.prefix_sum[r2 + 1][c2 + 1] \
             - self.prefix_sum[r1][c2 + 1] \
             - self.prefix_sum[r2 + 1][c1] \
             + self.prefix_sum[r1][c1]

        if sums != 10:
            return False

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
        best_move_for_me = (-1, -1, -1, -1)
        max_lookahead_score = -float('inf') 

        my_possible_moves = self._find_all_valid_moves()

        if not my_possible_moves:
            return (-1, -1, -1, -1)

        for my_move in my_possible_moves:
            # 1. 내 수를 시뮬레이션하기 위한 가상 게임 생성
            virtual_game_after_my_move = copy.deepcopy(self)
            virtual_game_after_my_move.updateMove(*my_move, isMyMove=True)
            
            # 2. 상대방의 최선 반격 예측
            opponent_best_reply = virtual_game_after_my_move._find_best_opponent_move()
            
            # 3. 상대방 반격까지 완료된 최종 가상 게임 생성
            final_virtual_game = copy.deepcopy(virtual_game_after_my_move)
            if opponent_best_reply != (-1, -1, -1, -1):
                final_virtual_game.updateMove(*opponent_best_reply, isMyMove=False)
            
            # --- 평가 시작 ---
            # 3-1. 기본 유불리 점수 (내 최종영역 - 상대 최종영역)
            current_lookahead_score = final_virtual_game._evaluate_board_state()
            
            # 3-2. [전략 수정] 강화된 위치 보너스
            positional_bonus = self._get_positional_bonus(my_move)
            current_lookahead_score += positional_bonus
            
            # 3-3. [전략 수정] 영토 침략 페널티 계산
            invasion_penalty = self._calculate_invasion_penalty(virtual_game_after_my_move, opponent_best_reply)
            current_lookahead_score += invasion_penalty

            if current_lookahead_score > max_lookahead_score:
                max_lookahead_score = current_lookahead_score
                best_move_for_me = my_move
                
        return best_move_for_me

    def _find_all_valid_moves(self):
        moves = []
        for r in range(self.rows):
            for c in range(self.cols):
                for r2 in range(r, self.rows):
                    for c2 in range(c, self.cols):
                        if self.isValid(r, c, r2, c2):
                            moves.append((r, c, r2, c2))
        return moves

    def _find_best_opponent_move(self):
        best_move = (-1, -1, -1, -1)
        max_score = -1
        
        possible_moves = self._find_all_valid_moves()
        for move in possible_moves:
            score = self.calculate_hybrid_score(*move, isMyMove=False)
            if score > max_score:
                max_score = score
                best_move = move
        return best_move

    def calculate_hybrid_score(self, r1, c1, r2, c2, isMyMove):
        stolen_cells = 0
        claimed_cells = 0
        
        my_current_area = self.my_area if isMyMove else self.opp_area
        opp_current_area = self.opp_area if isMyMove else self.my_area
        
        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                if opp_current_area[r][c]:
                    stolen_cells += 1
                elif not my_current_area[r][c]:
                    claimed_cells += 1
        
        return (stolen_cells * 10) + claimed_cells

    def _evaluate_board_state(self):
        my_total_area = sum(row.count(True) for row in self.my_area)
        opp_total_area = sum(row.count(True) for row in self.opp_area)
        return my_total_area - opp_total_area

    def _get_positional_bonus(self, move):
        """[전략 수정] 영역 1~2칸의 가치에 해당하는 강력한 위치 보너스를 부여합니다."""
        r1, c1, r2, c2 = move
        rows, cols = self.rows, self.cols

        is_top_left = (r1 == 0 and c1 == 0)
        is_top_right = (r1 == 0 and c2 == cols - 1)
        is_bottom_left = (r2 == rows - 1 and c1 == 0)
        is_bottom_right = (r2 == rows - 1 and c2 == cols - 1)

        if is_top_left or is_top_right or is_bottom_left or is_bottom_right:
            return 2.0  # 꼭짓점 보너스 (영역 2칸 가치)
        if r1 == 0 or r2 == rows - 1 or c1 == 0 or c2 == cols - 1:
            return 1.0  # 가장자리 보너스 (영역 1칸 가치)
        return 0.0

    def _calculate_invasion_penalty(self, game_state_before_opp_move, opp_move):
        """[전략 수정] 상대의 반격으로 내 영토가 침략당했다면 강력한 페널티를 부여합니다."""
        if opp_move == (-1, -1, -1, -1):
            return 0.0

        r1, c1, r2, c2 = opp_move
        stolen_from_me = 0
        
        # 상대방의 움직임(opp_move) 범위 내에서, 원래 내 땅이었던 칸의 수를 셉니다.
        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                # 'game_state_before_opp_move'는 내 수는 반영됐지만 상대 수는 반영되기 전의 상태입니다.
                if game_state_before_opp_move.my_area[r][c]:
                    stolen_from_me += 1
        
        # 1칸 뺏길 때마다 -2점씩 부과하여, AI가 영토를 잃는 것을 극도로 꺼리게 만듭니다.
        return stolen_from_me * -2.0

    def updateOpponentAction(self, action, _time):
        self.updateMove(*action, isMyMove=False)

    def updateMove(self, r1, c1, r2, c2, isMyMove):
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
    game = None
    first = False

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
