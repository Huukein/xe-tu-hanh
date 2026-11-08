# xe-tu-hanh
import time
from machine import PWM, Pin

# --- CẤU HÌNH ĐỘNG CƠ (L298N) ---
in1 = Pin(32, Pin.OUT)  # Đổi từ 34 sang 32 vì GPIO 34 chỉ là chân Input
in2 = Pin(25, Pin.OUT)
in3 = Pin(33, Pin.OUT)
in4 = Pin(26, Pin.OUT)

# Tốc độ động cơ (ENA, ENB dùng các chân GPIO dự phòng)
ena = PWM(Pin(4, Pin.OUT), freq=1000)
enb = PWM(Pin(16, Pin.OUT), freq=1000)

SPEED = 700  # Tốc độ từ 0 đến 1023


def set_speed(speed_left, speed_right):
  try:
    ena.duty(speed_left)
    enb.duty(speed_right)
  except:
    pass


def move_forward():
  in1.value(1)
  in2.value(0)
  in3.value(1)
  in4.value(0)
  set_speed(SPEED, SPEED)


def move_backward():
  in1.value(0)
  in2.value(1)
  in3.value(0)
  in4.value(1)
  set_speed(SPEED, SPEED)


def turn_left():
  in1.value(0)
  in2.value(1)
  in3.value(1)
  in4.value(0)
  set_speed(SPEED, SPEED)


def turn_right():
  in1.value(1)
  in2.value(0)
  in3.value(0)
  in4.value(1)
  set_speed(SPEED, SPEED)


def stop():
  in1.value(0)
  in2.value(0)
  in3.value(0)
  in4.value(0)
  set_speed(0, 0)


# --- 2 CẢM BIẾN TCRT5000 (DO) ---
# ir_left: GPIO 14, ir_right: GPIO 35 (Input-only phù hợp nhận tín hiệu DO)
# Quy ước: 0 = trắng bóng, 1 = đen
ir_left = Pin(14, Pin.IN)
ir_right = Pin(35, Pin.IN)

# --- CẢM BIẾN SIÊU ÂM HC-SR04 ---
trig = Pin(13, Pin.OUT)
echo = Pin(34, Pin.IN)  # GPIO 34 dùng làm echo (Input-only)


def get_distance():
  trig.value(0)
  time.sleep_us(2)
  trig.value(1)
  time.sleep_us(10)
  trig.value(0)

  timeout = 30000
  start_time = time.ticks_us()

  while echo.value() == 0:
    if time.ticks_diff(time.ticks_us(), start_time) > timeout:
      return 999

  t1 = time.ticks_us()
  while echo.value() == 1:
    if time.ticks_diff(time.ticks_us(), t1) > timeout:
      return 999
  t2 = time.ticks_us()

  duration = time.ticks_diff(t2, t1)
  distance = (duration * 0.0343) / 2
  return round(distance, 1)


# --- SERVO SG90 (PWM trên GPIO 27) ---
servo_pin = PWM(Pin(27, Pin.OUT), freq=50)


def set_servo_angle(angle):
  # angle: 0 (phải), 90 (giữa), 180 (trái)
  duty = int(26 + (angle / 180.0) * 97)
  servo_pin.duty(duty)
  time.sleep_ms(300)


# --- LOGIC NÉ VẬT CẢN (QUÉT TRÁI 90 ĐỘ -> PHẢI 180 ĐỘ) ---
def avoid_obstacle():
  stop()
  time.sleep_ms(200)

  # 1. Xoay 90 độ sang trái (tương ứng góc 180 độ của servo) để đo phía bên trái
  set_servo_angle(180)
  dist_left = get_distance()

  # 2. Xoay tiếp sang phải (từ trái qua phải là 180 độ, tương ứng góc 0 độ của servo) để đo bên phải
  set_servo_angle(0)
  dist_right = get_distance()

  # Trả servo về vị trí chính giữa (90 độ)
  set_servo_angle(90)

  # 3. Lách qua vật cản dựa vào khoảng cách trống hơn
  if dist_left > dist_right:
    # Lách qua bên trái
    turn_left()
    time.sleep_ms(500)
    move_forward()
    time.sleep_ms(600)
    turn_right()
    time.sleep_ms(500)
    move_forward()
    time.sleep_ms(600)
    turn_right()
    time.sleep_ms(400)
  else:
    # Lách qua bên phải
    turn_right()
    time.sleep_ms(500)
    move_forward()
    time.sleep_ms(600)
    turn_left()
    time.sleep_ms(500)
    move_forward()
    time.sleep_ms(600)
    turn_left()
    time.sleep_ms(400)

  stop()


# --- CHƯƠNG TRÌNH CHÍNH ---
def main():
  set_servo_angle(90)  # Đưa servo về giữa
  time.sleep(1)

  SAFE_DISTANCE = 15.0  # Khoảng cách phát hiện vật cản (cm)

  while True:
    # 1. Kiểm tra vật cản phía trước bằng HC-SR04
    dist_front = get_distance()
    if dist_front < SAFE_DISTANCE:
      avoid_obstacle()

    # 2. Đọc giá trị cảm biến dò line (0: trắng bóng, 1: đen)
    left_val = ir_left.value()
    right_val = ir_right.value()

    # 3. NHẬN BIẾT ĐÍCH: Nếu cả 2 cảm biến đều chạm vạch đen (1 và 1)
    if left_val == 1 and right_val == 1:
      stop()
      print("Đã đến đích!")
      while True:
        pass  # Dừng hẳn chương trình tại đích

    # 4. Logic dò line tiêu chuẩn (0: trắng bóng, 1: đen)
    # Cả 2 đều trên nền trắng bóng (0 và 0): Đi thẳng
    if left_val == 0 and right_val == 0:
      move_forward()
    # Cảm biến trái lệch vào line đen (1), phải ở nền trắng (0) -> Bẻ lái trái
    elif left_val == 1 and right_val == 0:
      turn_left()
    # Cảm biến phải lệch vào line đen (1), trái ở nền trắng (0) -> Bẻ lái phải
    elif left_val == 0 and right_val == 1:
      turn_right()
    else:
      move_forward()

    time.sleep_ms(10)


if __name__ == "__main__":
  main()
