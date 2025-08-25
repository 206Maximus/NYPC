import sys
import copy
import math

class Game:

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
                self.prefix_sum[r+1][c+1] = self.prefix_sum[r][c+1] + self.prefix_sum[r+1][c] - self.prefix_sum[r][c] + self.board[r][c]

    def isValid(self, r1, c1, r2, c2):
        if not (0 <= r1 < self.rows and 0 <= c1 < self.cols and r1 <= r2 < self.rows and c1 <= c2 < self.cols):
            return False
        sums = self.prefix_sum[r2 + 1][c2 + 1] - self.prefix_sum[r1][c2 + 1] - self.prefix_sum[r2 + 1][c1] + self.prefix_sum[r1][c1]
        if sums != 10: return False
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
        """Minimax 탐색을 시작하여 최적의 수를 계산합니다."""
        best_move = (-1, -1, -1, -1)
        max_eval = -math.inf
        alpha = -math.inf
        beta = math.inf

        # 탐색 깊이. 3은 (나의 수 -> 상대 수 -> 나의 수) 3단계 앞을 의미합니다.
        # 시간 초과가 발생하면 2 또는 1로 줄여서 안정성을 확보할 수 있습니다.
        SEARCH_DEPTH = 3

        my_possible_moves = self._find_all_valid_moves()
        if not my_possible_moves:
            return best_move
            
        # 유망한 수부터 탐색하기 위해 휴리스틱으로 정렬 (알파-베타 프루닝 효율 극대화)
        sorted_moves = sorted(my_possible_moves, key=lambda m: self._get_heuristic_score(m, isMyMove=True), reverse=True)

        for move in sorted_moves:
            virtual_game = copy.deepcopy(self)
            virtual_game.updateMove(*move, isMyMove=True)
            
            # 다음은 상대방 턴(Minimizing Player)이므로 is_maximizing_player=False로 호출
            evaluation = self._minimax(virtual_game, SEARCH_DEPTH - 1, alpha, beta, False)
            
            if evaluation > max_eval:
                max_eval = evaluation
                best_move = move
            
            alpha = max(alpha, evaluation)

        return best_move

    def _minimax(self, game_state, depth, alpha, beta, is_maximizing_player):
        """Minimax 알고리즘과 알파-베타 프루닝으로 최적의 수를 탐색합니다."""
        if depth == 0 or not game_state._find_all_valid_moves():
            return game_state._evaluate_board_state()

        possible_moves = game_state._find_all_valid_moves()
        sorted_moves = sorted(possible_moves, key=lambda m: game_state._get_heuristic_score(m, isMyMove=is_maximizing_player), reverse=is_maximizing_player)

        if is_maximizing_player: # 나의 턴 (점수 극대화)
            max_eval = -math.inf
            for move in sorted_moves:
                virtual_game = copy.deepcopy(game_state)
                virtual_game.updateMove(*move, isMyMove=True)
                evaluation = self._minimax(virtual_game, depth - 1, alpha, beta, False)
                max_eval = max(max_eval, evaluation)
                alpha = max(alpha, evaluation)
                if beta <= alpha:
                    break # Alpha Cutoff
            return max_eval
        else: # 상대방 턴 (점수 최소화)
            min_eval = math.inf
            for move in sorted_moves:
                virtual_game = copy.deepcopy(game_state)
                virtual_game.updateMove(*move, isMyMove=False)
                evaluation = self._minimax(virtual_game, depth - 1, alpha, beta, True)
                min_eval = min(min_eval, evaluation)
                beta = min(beta, evaluation)
                if beta <= alpha:
                    break # Beta Cutoff
            return min_eval

    def _get_heuristic_score(self, move, isMyMove):
        """알파-베타 프루닝의 효율을 높이기 위한 휴리스틱 평가 함수."""
        return self.calculate_score_by_request(*move, isMyMove=isMyMove)
    
    def calculate_score_by_request(self, r1, c1, r2, c2, isMyMove):
        score = 0
        my_current_area = self.my_area if isMyMove else self.opp_area
        opp_current_area = self.opp_area if isMyMove else self.my_area

        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                if opp_current_area[r][c]:
                    score += 1.5
                elif not my_current_area[r][c]:
                    is_edge = (r == 0 or r == self.rows - 1 or c == 0 or c == self.cols - 1)
                    if is_edge:
                        score += 2.2
                    else:
                        score += 0.75
        return score

    def _find_all_valid_moves(self):
        moves = []
        for r in range(self.rows):
            for c in range(self.cols):
                for r2 in range(r, self.rows):
                    for c2 in range(c, self.cols):
                        if self.isValid(r, c, r2, c2):
                            moves.append((r, c, r2, c2))
        return moves
    
    def _evaluate_board_state(self):
        """탐색의 마지막 노드에서 보드의 최종 가치를 평가 (나의 영역 - 상대 영역)"""
        my_total_area = sum(row.count(True) for row in self.my_area)
        opp_total_area = sum(row.count(True) for row in self.opp_area)
        return my_total_area - opp_total_area

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
            # board = [list(map(int, list(row))) for row in board_data]
            board = [list(map(int, list(row))) for row in param]
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
