#include <Wire.h>
#include <I2C_RTC.h>
#include <LiquidCrystal.h>

#define RELAY_PIN A3
#define RESET_BUTTON A2
#define RING_DURATION_SHORT 3000  // 3 секунди
#define RING_DURATION_DELAY 1000  // 1 секунда
#define RING_DURATION_LONG 5000   // 5 секунди

#define BUTTON_PIN A0

static DS3231 RTC;
LiquidCrystal lcd(8, 9, 4, 5, 6, 7);

enum Button {
  NONE,
  RIGHT,
  UP,
  DOWN,
  LEFT,
  SELECT
};

void printDayName(int day) {
  switch (day) {
    case 0:
      lcd.print("Vtornik");
      break;
    case 1:
      lcd.print("Srqda");
      break;
    case 2:
      lcd.print("Chetvurtuk");
      break;
    case 3:
      lcd.print("Petuk");
      break;
    case 4:
      lcd.print("Subota");
      break;
    case 5:
      lcd.print("Nedelq");
      break;
    case 6:
      lcd.print("Ponedelnik");
      break;
    default:
      Serial.println("Невалиден ден");
      break;
  }
}

Button getButton() {
  int value = analogRead(BUTTON_PIN);
  if (value < 900)
    if (value < 50) return RIGHT;
  if (value < 200) return UP;
  if (value < 400) return DOWN;
  if (value < 600) return LEFT;
  if (value < 900) return SELECT;
  return NONE;
}

struct BellTime {
  int hour;
  int minute;
  int type;
};

BellTime bellSchedule[] = {
  { 7, 40, 2 }, { 8, 20, 1 }, { 8, 35, 1 }, { 8, 40, 2 },  // 0, 1, 2, 3
  { 9, 20, 1 },
  { 9, 25, 1 },
  { 9, 30, 2 },
  { 10, 10, 1 },  // 4, 5, 6, 7
  { 10, 15, 1 },
  { 10, 20, 2 },
  { 11, 0, 1 },
  { 11, 5, 1 },  // 8, 9, 10, 11
  { 11, 10, 2 },
  { 11, 50, 1 },
  { 11, 55, 1 },
  { 12, 0, 2 },
  { 12, 40, 1 },  // 12, 13 , 14, 15
  { 12, 45, 1 },
  { 12, 50, 2 },
  { 13, 40, 1 },
  { 13, 45, 2 },
  { 14, 5, 2 }  // 19
};

const BellTime bellScheduleRes[] = {
  { 7, 40, 2 }, { 8, 20, 1 }, { 8, 35, 1 }, { 8, 40, 2 },  // 0, 1, 2, 3
  { 9, 20, 1 },
  { 9, 25, 1 },
  { 9, 30, 2 },
  { 10, 10, 1 },  // 4, 5, 6, 7
  { 10, 15, 1 },
  { 10, 20, 2 },
  { 11, 0, 1 },
  { 11, 5, 1 },  // 8, 9, 10, 11
  { 11, 10, 2 },
  { 11, 50, 1 },
  { 11, 55, 1 },
  { 12, 0, 2 },
  { 12, 40, 1 },  // 12, 13 , 14, 15
  { 12, 45, 1 },
  { 12, 50, 2 },
  { 13, 40, 1 },
  { 13, 45, 2 },
  { 14, 5, 2 }  // 19
};
const int bellCount = sizeof(bellSchedule) / sizeof(BellTime);

int currHour = -2;
bool setSchoolBell = false;
int removedMinutes = 0;  // Натрупани премахнати минути
int addedMinutes = 0;    // Върнати минути чрез Right

int editingHour = 0;
int editingMinute = 0;
bool isEditing = false;
uint8_t dayOfWeek = 0;

bool settings = false;
bool setSwitchHours = false;
int save = 0;
int func = 0;
int setHours = 0;
int setMinutes = 0;
int setRealTime = false;
bool setDate = false;

void setup() {
  Serial.begin(9600);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, HIGH);

  lcd.begin(16, 2);
  Wire.begin();
  RTC.begin();

  //RTC.setDate(25, 2, 2025);  // Ден, месец, година
  // RTC.setTime(18, 11, 0);    // Час, минути, секунди
}

void loop() {
  Button pressed = getButton();

  if (setRealTime) {
    if (setSwitchHours) {
      lcd.clear();
      lcd.setCursor(0, 0);
      if (setHours < 10) lcd.print("0");
      lcd.print(setHours);
      lcd.print(":");
      if (setMinutes < 10) lcd.print("0");
      lcd.print(setMinutes);
      lcd.setCursor(0, 1);
      lcd.print("Chas");
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      if (setHours < 10) lcd.print("0");
      lcd.print(setHours);
      lcd.print(":");
      if (setMinutes < 10) lcd.print("0");
      lcd.print(setMinutes);
      lcd.setCursor(0, 1);
      lcd.print("Minuti");
    }
  }
  if (setDate) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Den:");
    printDayName(dayOfWeek);
  }

  if (pressed != NONE) {
    Serial.print("Button Pressed: ");

    switch (pressed) {
      case SELECT:
        {
          Serial.println(save);
          if (func == 1) {
            setRealTime = true;
            if (save == 2) {
              setRealTime = false;
              setDate = true;
            }
            if (save == 3) {
              save = 0;
              setRealTime = false;
              setDate = false;
              RTC.setTime(setHours, setMinutes, 0);  // Час, минути, секунди
              setHours = 0;
              setMinutes = 0;
              lcd.clear();
              lcd.setCursor(0, 0);
              lcd.print("Zapazvane.");
              delay(2000);
              settings = false;
            }
            save += 1;
          } else if (func == 2) {
            setSchoolBell = true;
          } else if (func == 3) {
            settings = false;
            pressed = Button::NONE;
            break;
          } else {
            lcd.clear();
            lcd.setCursor(0, 0);
            lcd.print("Izberi: ");
            settings = true;
          }
          break;
        }
      case UP:
        {
          Serial.println("Up");
          if (setRealTime) {
            if (!setSwitchHours) {
              if (setHours < 23) {
                ++setHours;
              } else {
                setHours = 0;
              }
            } else {
              if (setMinutes > 59) {
                setMinutes = 0;
              } else {
                ++setMinutes;
              }
            }
            pressed = Button::NONE;
            break;
          }
          if (setDate) {
            ++dayOfWeek;
            break;
          }

          if (func == 4) {
            func = 0;
          }
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Izberi:");
          lcd.print("Chas/data");
          if (func == 1) {
            lcd.clear();
            lcd.setCursor(0, 0);
            lcd.print("Izberi:");
            lcd.print("Chasove");
          }
          if (func == 2) {
            lcd.clear();
            lcd.setCursor(0, 0);
            lcd.print("Izberi:");
            lcd.print("Izlizane");
          }
          func++;
          while (getButton() == UP)
            ;
        }
        break;
      case DOWN:
        {
          Serial.println("Down");
          if (setRealTime) {
            if (!setSwitchHours) {
              if (setHours <= 0) {
                setHours = 23;
              } else {
                --setHours;
              }
            } else {
              if (setMinutes <= 0) {
                setMinutes = 59;
              } else {
                --setMinutes;
              }
            }
            pressed = Button::NONE;
            break;
          }
          if (setDate) {
            --dayOfWeek;
            break;
          }
          while (getButton() == DOWN)
            ;
        }
        break;
      case RIGHT:
        {
          if (!setSwitchHours) {
            setSwitchHours = true;
          }

          while (getButton() == RIGHT)
            ;
        }
        break;
      case LEFT:
        {
          if (setSwitchHours) {
            setSwitchHours = false;
          }

          while (getButton() == LEFT)
            ;
        }
        break;
    }

    delay(200);  // Debounce

    while (setSchoolBell) {
      Button press = getButton();

      switch (press) {
        case LEFT:
          {
            Serial.println("Left");

            if (removedMinutes < 20) {
              editingMinute -= 5;
              removedMinutes += 5;
              addedMinutes = 0;  // След премахване, нулираме добавените минути

              if (editingMinute < 0) {
                if (editingHour > 0) {
                  editingHour--;
                  editingMinute = 55;
                } else {
                  editingMinute = 0;
                }
              }

              lcd.setCursor(0, 1);
              lcd.print("Izberi: ");
              lcd.print(editingHour);
              lcd.print(":");
              if (editingMinute < 10) lcd.print("0");
              lcd.print(editingMinute);
            }

            while (getButton() == LEFT)
              ;
          }
          break;

        case RIGHT:
          {
            Serial.println("Right");

            if (removedMinutes > addedMinutes) {
              editingMinute += 5;
              addedMinutes += 5;

              if (editingMinute > 59) {
                editingHour++;
                editingMinute = 0;
              }

              lcd.setCursor(0, 1);
              lcd.print("Izberi: ");
              lcd.print(editingHour);
              lcd.print(":");
              if (editingMinute < 10) lcd.print("0");
              lcd.print(editingMinute);
            }

            while (getButton() == RIGHT)
              ;
          }
          break;

        case UP:
          {
            Serial.println("Up");

            // Смяна на текущия час за редакция
            if (currHour < 19) {
              if (currHour < 13) {
                currHour += 3;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              } else if (currHour >= 13) {
                currHour += 2;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              }
              Serial.println(bellSchedule[currHour].hour);
              Serial.println(bellSchedule[currHour].minute);
            }

            lcd.setCursor(0, 1);
            lcd.print("Izberi: ");
            lcd.print(editingHour);
            lcd.print(":");
            if (editingMinute < 10) lcd.print("0");
            lcd.print(editingMinute);

            while (getButton() == UP)
              ;
          }
          break;

        case DOWN:
          {
            Serial.println("Down");

            if (currHour > 1) {
              if (currHour > 13) {
                currHour -= 2;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              } else if (currHour <= 13) {
                currHour -= 3;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              }
            }

            lcd.setCursor(0, 1);
            lcd.print("Izberi: ");
            lcd.print(editingHour);
            lcd.print(":");
            if (editingMinute < 10) lcd.print("0");
            lcd.print(editingMinute);

            while (getButton() == DOWN)
              ;
          }
          break;

        case SELECT:
          {
            Serial.println("Zapazeno!");

            while (getButton() == SELECT)
              ;

            if (setSchoolBell) {
              setSchoolBell = false;
              lcd.clear();
              lcd.print("Zapazeno!");
              delay(1000);
              lcd.clear();

              // Запазване на промените
              bellSchedule[currHour].hour = editingHour;
              bellSchedule[currHour].minute = editingMinute;

              // Променяме всички следващи часове
              int totalChange = removedMinutes - addedMinutes;

              for (int i = currHour + 1; i < bellCount; i++) {
                bellSchedule[i].minute -= totalChange;

                if (bellSchedule[i].minute < 0) {
                  if (bellSchedule[i].hour > 0) {
                    bellSchedule[i].hour--;
                    bellSchedule[i].minute += 60;
                  } else {
                    bellSchedule[i].minute = 0;
                  }
                }
              }

              removedMinutes = 0;
              addedMinutes = 0;
              isEditing = false;
              currHour = -2;
            }
          }
          break;

          delay(500);
      }
    }
  }


  if (!settings) {
    displayTime();
    if (dayOfWeek != 0 || dayOfWeek != 6) {
      checkBellSchedule();
    }
  }

  if (digitalRead(RESET_BUTTON) == HIGH) {
    for (int i = 0; i < 19; i++) {
      bellSchedule[i].hour = bellScheduleRes[i].hour;
      bellSchedule[i].minute = bellScheduleRes[i].minute;
    }
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Pochistvane na");
    lcd.setCursor(0, 1);
    lcd.print("nastroikite");
    delay(3000);
  }
}

void displayTime() {
  lcd.setCursor(0, 0);
  lcd.print("Chas: ");
  lcd.print(RTC.getHours());
  lcd.print(":");
  if (RTC.getMinutes() < 10) lcd.print("0");
  lcd.print(RTC.getMinutes());
}

void ringBell(int type) {
  if (type == 1) {
    digitalWrite(RELAY_PIN, LOW);
    delay(RING_DURATION_LONG);
    digitalWrite(RELAY_PIN, HIGH);
  } else if (type == 2) {
    digitalWrite(RELAY_PIN, LOW);
    delay(RING_DURATION_SHORT);
    digitalWrite(RELAY_PIN, HIGH);
    delay(RING_DURATION_DELAY);
    digitalWrite(RELAY_PIN, LOW);
    delay(RING_DURATION_SHORT);
    digitalWrite(RELAY_PIN, HIGH);
  }
}

void checkBellSchedule() {
  int currentHour = RTC.getHours();
  int currentMinute = RTC.getMinutes();

  for (int i = 0; i < bellCount; i++) {
    if (bellSchedule[i].hour == currentHour && bellSchedule[i].minute == currentMinute) {
      ringBell(bellSchedule[i].type);
      delay(60000);  // Изчакване 1 минута за избягване на повторения
      break;
    }
  }
}
