import time
from machine import PWM, Pin

# ============================================================
#  CẤU HÌNH CHÂN
# ============================================================
in1 = Pin(27, Pin.OUT, value=0)  # bánh phải (tiến)
in2 = Pin(19, Pin.OUT, value=0)  # bánh phải (lùi)
in3 = Pin(25, Pin.OUT, value=0)  # bánh trái (tiến)
in4 = Pin(33, Pin.OUT, value=0)  # bánh trái (lùi)

ir_left = Pin(39, Pin.IN)        # GPIO34/39 là chân chỉ đọc
ir_right = Pin(34, Pin.IN)

ENA_PIN = 18                     # ENA điều khiển bánh PHẢI
ENB_PIN = 23                     # ENB điều khiển bánh TRÁI
PWM_FREQ = 5000

ena = PWM(Pin(ENA_PIN), freq=PWM_FREQ, duty=0)
enb = PWM(Pin(ENB_PIN), freq=PWM_FREQ, duty=0)

# ============================================================
#  THÔNG SỐ CẦN HIỆU CHỈNH
# ============================================================
TEST_MOTOR = False
LEFT_REVERSED = False
RIGHT_REVERSED = True
ON_LINE = 1                      # Giá trị cảm biến khi nằm TRÊN vạch
START_DELAY_S = 2
DEBUG = True

BASE_SPEED = 800                 # Tốc độ đi thẳng (0-1023)
TURN_SPEED = 750                 # Tốc độ quay tại chỗ khi rẽ

# --- RẼ "CHỐT" ---
# Chỉ cần cảm biến nháy MỘT cái là xe chốt rẽ:
#   1) Quay đủ TURN_TIME_MS, BỎ QUA hoàn toàn cảm biến
#   2) Hết thời gian đó mà cảm biến còn đen (góc dày) -> quay tiếp
#      cho đến khi CẢ HAI cảm biến hết đen (vạch nằm giữa lại)
#   3) Phanh ngắn rồi đi thẳng
TURN_TIME_MS = 50               # Thời gian quay tối thiểu (hết giờ mà còn đen thì quay tiếp)
TURN_PAUSE_MS = 0                # Dừng trước khi rẽ (ms). 1000 = dừng 1 giây
TURN_BRAKE_MS = 30               # Phanh ngắn sau khi rẽ xong để xe không trôi
CORNER_TIMEOUT_MS = 5000         # Quay quá lâu mà chưa thoát -> dừng cho an toàn

# --- CHỐNG RẼ LIÊN TIẾP (chống chệch line) ---
# Sau khi rẽ xong, trong SETTLE_MS tiếp theo xe KHÔNG chốt rẽ nữa mà chỉ nắn nhẹ
# (bánh trong đứng yên) cho tới khi hết đen. Nhờ vậy xe quay hơi thừa thì chỉ
# nắn lại chứ không quay thêm một cú nữa.
SETTLE_MS = 500                  # Thời gian "ổn định" sau mỗi lần rẽ
FINE_HOLD_MS = 80                # Giữ nắn nhẹ thêm sau khi cảm biến hết đen
GENTLE_SPEED = 750               # Tốc độ bánh ngoài khi nắn nhẹ

# --- VẠCH NGANG ---
STOP_AT_CROSS = True             # Đang đi thẳng, 2 cảm biến cùng đen -> dừng hẳn


# ============================================================
#  ĐỘNG CƠ
# ============================================================
def set_wheels(left, right):
    """left/right trong khoảng -1023..1023 (âm = lùi)."""
    if LEFT_REVERSED:
        left = -left
    if RIGHT_REVERSED:
        right = -right

    in3.value(1 if left > 0 else 0)
    in4.value(1 if left < 0 else 0)
    in1.value(1 if right > 0 else 0)
    in2.value(1 if right < 0 else 0)

    enb.duty(min(abs(int(left)), 1023))
    ena.duty(min(abs(int(right)), 1023))


def stop_car():
    set_wheels(0, 0)


def go_straight():
    set_wheels(BASE_SPEED, BASE_SPEED)


def pivot(direction):
    # Quay tại chỗ: rẽ trái = bánh trái lùi + bánh phải tiến, rẽ phải ngược lại
    if direction == "left":
        set_wheels(-TURN_SPEED, TURN_SPEED)
    else:
        set_wheels(TURN_SPEED, -TURN_SPEED)


def gentle(direction):
    # Nắn nhẹ: bánh trong đứng yên, bánh ngoài chạy tiến
    if direction == "left":
        set_wheels(0, GENTLE_SPEED)
    else:
        set_wheels(GENTLE_SPEED, 0)


def test_motors():
    steps = (
        ("Bánh TRÁI tiến", (BASE_SPEED, 0)),
        ("Bánh PHẢI tiến", (0, BASE_SPEED)),
        ("CẢ HAI tiến", (BASE_SPEED, BASE_SPEED)),
        ("CẢ HAI lùi", (-BASE_SPEED, -BASE_SPEED)),
    )
    print("NHẤC BÁNH XE KHỎI MẶT ĐẤT! Bắt đầu sau 3 giây...")
    time.sleep(3)
    for name, (l, r) in steps:
        print(name)
        set_wheels(l, r)
        time.sleep(2)
        stop_car()
        time.sleep(1)
    print("Xong test động cơ.")


# ============================================================
#  CẢM BIẾN (lọc nhiễu nhẹ)
# ============================================================
def raw_sensors():
    return (ir_left.value() == ON_LINE, ir_right.value() == ON_LINE)


def read_sensors(prev):
    """Đọc 2 lần liên tiếp, giống nhau mới nhận; khác nhau thì giữ giá trị cũ."""
    a = raw_sensors()
    time.sleep_us(300)
    b = raw_sensors()
    return b if a == b else prev


# ============================================================
#  CHƯƠNG TRÌNH CHÍNH
# ============================================================
def main():
    stop_car()

    if TEST_MOTOR:
        test_motors()
        return

    print("Chờ", START_DELAY_S, "giây trước khi xuất phát...")
    time.sleep(START_DELAY_S)
    print("Bắt đầu dò line")

    sensors = (False, False)
    mode = "run"                # "run" = đi thẳng/nắn nhẹ, "turn" = đang rẽ chốt
    direction = None            # hướng rẽ gần nhất
    turn_start = time.ticks_ms()
    settle_until = turn_start   # trước mốc này: chỉ nắn nhẹ, không chốt rẽ
    fine_until = turn_start     # giữ nắn nhẹ tới mốc này
    fine_dir = None
    last_print = turn_start

    while True:
        sensors = read_sensors(sensors)
        l, r = sensors
        now = time.ticks_ms()

        if mode == "turn":
            elapsed = time.ticks_diff(now, turn_start)
            if elapsed < TURN_TIME_MS:
                pass                                    # BỎ QUA cảm biến, cứ quay
            elif not l and not r:
                # Đã quay đủ và vạch đã nằm giữa -> phanh ngắn, đi thẳng
                if TURN_BRAKE_MS > 0:
                    stop_car()
                    time.sleep_ms(TURN_BRAKE_MS)
                mode = "run"
                fine_dir = None
                now = time.ticks_ms()
                settle_until = time.ticks_add(now, SETTLE_MS)
                go_straight()
            elif elapsed > CORNER_TIMEOUT_MS:
                print("Quay quá lâu -> dừng")
                break
            # else: còn đen (góc dày) -> quay tiếp

        else:
            settling = time.ticks_diff(settle_until, now) > 0

            if l and r:
                if settling:
                    gentle(direction or "left")         # chưa ra khỏi góc: nắn theo hướng cũ
                elif STOP_AT_CROSS:
                    print("Gặp vạch ngang -> dừng")
                    break
                else:
                    go_straight()

            elif l or r:
                d = "left" if l else "right"
                if settling:
                    # Vừa rẽ xong: chỉ NẮN NHẸ, không chốt rẽ nữa
                    fine_dir = d
                    fine_until = time.ticks_add(now, FINE_HOLD_MS)
                    gentle(d)
                else:
                    # Cảm biến nháy MỘT cái -> chốt rẽ
                    direction = d
                    if TURN_PAUSE_MS > 0:
                        stop_car()
                        time.sleep_ms(TURN_PAUSE_MS)
                    pivot(direction)
                    turn_start = time.ticks_ms()
                    mode = "turn"

            else:
                if fine_dir is not None and time.ticks_diff(fine_until, now) > 0:
                    gentle(fine_dir)                    # giữ nắn thêm một chút
                else:
                    fine_dir = None
                    go_straight()

        if DEBUG and time.ticks_diff(now, last_print) >= 500:
            last_print = now
            print("L:", int(l), "R:", int(r), "| mode:", mode, "| huong:", direction)

        time.sleep_ms(1)


if __name__ == "__main__":
    try:
        main()
    finally:
        stop_car()
        ena.duty(0)
        enb.duty(0)
