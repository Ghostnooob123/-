```c++
#include <Wire.h>                 (Библиотека за I2C комуникация)
#include <I2C_RTC.h>             (Библиотека за работа с RTC модул - часовник)
#include <LiquidCrystal.h>       (Библиотека за LCD дисплей)

#define RELAY_PIN A3             (Пин за управление на релето)
#define RESET_BUTTON A2          (Пин за бутон за нулиране - не се използва тук)
#define RING_DURATION_SHORT 3000 (Кратък звънец - 3 секунди)
#define RING_DURATION_DELAY 1000 (Закъснение между два звънеца - 1 секунда)
#define RING_DURATION_LONG 5000  (Дълъг звънец - 5 секунди)

#define BUTTON_PIN A0            (Аналогов пин за бутоните на LCD Shield-а)

static DS3231 RTC;               (Инициализиране на RTC часовника)
LiquidCrystal lcd(8, 9, 4, 5, 6, 7); (Настройка на пиновете за LCD дисплея)





enum Button {                    (Тип за бутоните на LCD Shield-а)
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
      lcd.print("Vtornik");     (Вторник)
      break;
    case 1:
      lcd.print("Srqda");       (Сряда)
      break;
    case 2:
      lcd.print("Chetvurtuk");  (Четвъртък)
      break;
    case 3:
      lcd.print("Petuk");       (Петък)
      break;


    case 4:
      lcd.print("Subota");      (Събота)
      break;
    case 5:
      lcd.print("Nedelq");      (Неделя)
      break;
    case 6:
      lcd.print("Ponedelnik");  (Понеделник)
      break;
    default:
      Serial.println("Невалиден ден"); (Грешен номер на ден)
      break;
  }
}

Button getButton() {
  int value = analogRead(BUTTON_PIN); (Четене на аналоговата стойност от бутоните)
  if (value < 900)
    if (value < 50) return RIGHT;     (Десен бутон)
  if (value < 200) return UP;         (Горе)
  if (value < 400) return DOWN;       (Долу)
  if (value < 600) return LEFT;       (Ляво)
  if (value < 900) return SELECT;     (Select бутон)

return NONE;      (Няма натиснат бутон)
}

struct BellTime {          (Структура за един час на звънене)
  int hour;
  int minute;
  int type;       (Тип звънец: 1 - кратък, 2 - дълъг)
};

BellTime bellSchedule[] = {
  { 7, 40, 2 }, { 8, 20, 1 }, { 8, 35, 1 }, { 8, 40, 2 },
  { 9, 20, 1 }, { 9, 25, 1 }, { 9, 30, 2 },             
  { 10, 10, 1 }, { 10, 15, 1 }, { 10, 20, 2 },            
  { 11, 0, 1 }, { 11, 5, 1 }, { 11, 10, 2 },
  { 11, 50, 1 }, { 11, 55, 1 }, { 12, 0, 2 },
  { 12, 40, 1 }, { 12, 45, 1 }, { 12, 50, 2 },
  { 13, 40, 1 }, { 13, 45, 2 }, { 14, 5, 2 }            
}; (График на звънеца)








const BellTime bellScheduleRes[] = {
{ 7, 40, 2 }, { 8, 20, 1 }, { 8, 35, 1 }, { 8, 40, 2 }
{ 9, 20, 1 }, { 9, 25, 1 }, { 9, 30, 2 },
{ 10, 10, 1 }, { 10, 15, 1 }, { 10, 20, 2 },
{ 11, 0, 1 }, { 11, 5, 1 }, { 11, 10, 2 },
{ 11, 50, 1 }, { 11, 55, 1 }, { 12, 0, 2 },
{ 12, 40, 1 }, { 12, 45, 1 }, { 12, 50, 2 },
{ 13, 40, 1 }, { 13, 45, 2 }, { 14, 5, 2 }
};  (Резервно копие на графика)

const int bellCount = sizeof(bellSchedule) / sizeof(BellTime); (Брой звънци в графика)

int currHour = -2;  (Текущият час - използва се за следене на промяна)
bool setSchoolBell = false;      (Флаг дали сме в режим на звънене)
int removedMinutes = 0;      (Минутите, премахнати от началния час)
int addedMinutes = 0;            (Минутите, върнати чрез бутона RIGHT)
int editingHour = 0;             (Час за редакция)
int editingMinute = 0;           (Минута за редакция)
bool isEditing = false;          (Флаг дали сме в режим на редакция)
int dayOfWeek = 6;               (Ден от седмицата, по подразбиране - понеделник)
bool settings = false;           (Флаг за настройки)

bool setSwitchHours = false;     (Флаг за редакция на часове)
int save = 0;                    (Помощна променлива за състояния)
int func = 0;            (Друга помощна променлива за функционалности)
int setHours = 0;             (Час от реалното време при стартиране)
int setMinutes = 0;              (Минута от реалното време при стартиране)
int setRealTime = false;         (Флаг за настройка на реалното време)
bool setDate = false;            (Флаг за настройка на дата)

void setup() {
  Serial.begin(9600);            (Стартиране на серийната комуникация)
  pinMode(RELAY_PIN, OUTPUT);  (Настройка на релето като изход)
  digitalWrite(RELAY_PIN, HIGH); (По подразбиране релето е изключено)
  lcd.begin(16, 2);           (Инициализация на LCD дисплея 16x2)
  Wire.begin();                (Стартиране на I2C комуникация)
  RTC.begin();                 (Стартиране на RTC модула)
  //RTC.setDate(25, 2, 2025);    (Ръчно задаване на дата - ден, месец, година)
  //RTC.setTime(7, 41, 55);      (Ръчно задаване на време - час, минута, секунда)
  setHours = RTC.getHours();     (Вземане на текущия час от RTC)
  setMinutes = RTC.getMinutes(); (Вземане на текущата минута от RTC)
}

void loop() {
  Button pressed = getButton(); (Проверява кой бутон е натиснат)

  (Ако сме в режим на настройка на реалното време — показва текущите настройки за час и минути)
  if (setRealTime) {
    if (!setSwitchHours) { (Ако сме в режим за настройка на часа)
      lcd.clear();
      lcd.setCursor(0, 0);
      if (setHours < 10) lcd.print("0");
      lcd.print(setHours);
      lcd.print(":");
      if (setMinutes < 10) lcd.print("0");
      lcd.print(setMinutes);
      lcd.setCursor(0, 1);
      lcd.print("Chas"); (Показва "Час")
    } else { (В противен случай — настройка на минутите)
      lcd.clear();
      lcd.setCursor(0, 0);
      if (setHours < 10) lcd.print("0");
      lcd.print(setHours);
      lcd.print(":");
      if (setMinutes < 10) lcd.print("0");
      lcd.print(setMinutes);
      lcd.setCursor(0, 1);
      lcd.print("Minuti"); (Показва "Минути")
    }
    delay(200); (Забавяне, за да не мига екрана твърде бързо)
  }
  (Ако сме в режим на настройка на датата — показва деня от седмицата)
  if (setDate) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Den:");
    printDayName(dayOfWeek); (Показва текущия ден от седмицата)
    delay(200);
  }
  (Ако има натиснат бутон — обработва съответното действие)
  if (pressed != NONE) {
    Serial.print("Button Pressed: ");
    switch (pressed) {
      case SELECT:
        {
          Serial.println(save);
          (Обработка на бутона SELECT в зависимост от активната функция)
          if (func == 1) { (Ако сме в режим на настройка на час и дата)
            setRealTime = true;

            if (save == 2) {
              setRealTime = false;
              setDate = true;
            }
            if (save == 3) { (Записва новото време в RTC и излиза от настройката)
              save = 0;
              setRealTime = false;
              setDate = false;
              RTC.setTime(setHours, setMinutes, 0);
              setHours = RTC.getHours();
              setMinutes = RTC.getMinutes();
              lcd.clear();
              lcd.setCursor(0, 0);
              lcd.print("Zapazvane."); (Показва "Запазване.")
              delay(2000);
              settings = false;
              func = 0;
              dayOfWeek = 6;
            }
            save += 1;
          } else if (func == 2) { (Влизане в редактиране на училищния звънец)
            setSchoolBell = true;

            func = 0;
            lcd.clear();
          } else if (func == 3) { (Излизане от менюто за настройки)
            settings = false;
            pressed = Button::NONE;
            lcd.clear();
            func = 0;
            break;
          } else { (Показва меню за избор на настройки)
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
          (Обработка на бутон UP в зависимост от режима)
          if (setRealTime) { (Увеличава часа или минутите при настройка)
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
          if (setDate) { (Променя деня от седмицата)
            ++dayOfWeek;
            if (dayOfWeek > 6) {
              dayOfWeek = 0;
            }
            break;
          }


          (Навигация в менюто — преминава към следващата функция)
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
          while (getButton() == UP) ; (Изчаква бутонът да бъде пуснат)
        } break;

      case DOWN:
        {
          Serial.println("Down");
          (Обработка на бутон DOWN в зависимост от режима)
          if (setRealTime) { (Намаля часа или минутите при настройка)
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
          if (setDate) { (Променя деня от седмицата надолу)
            --dayOfWeek;

            if (dayOfWeek < 0) {
              dayOfWeek = 6;
            }
            break;
          }
          while (getButton() == DOWN) ; (Изчаква бутонът да бъде пуснат)
        }
        break;

      case RIGHT:
        {
          (Превключва режима на настройка между часове и минути)
          if (!setSwitchHours) {
            setSwitchHours = true;
          }
          while (getButton() == RIGHT) ; (Изчаква бутонът да бъде пуснат)
        }
        break;

      case LEFT:
        {
          (Превключва режима на настройка между минути и часове)

          if (setSwitchHours) {
            setSwitchHours = false;
          }
          while (getButton() == LEFT) ; (Изчаква бутонът да бъде пуснат)
        }
        break;
    }

    delay(200); (Забавяне за предотвратяване на многократно четене от бутона)

    (Цикъл за редактиране на графика на училищния звънец)
    while (setSchoolBell) {
      Button press = getButton();

      if (currHour > -2) {
        lcd.setCursor(0, 0);
        lcd.print("Promeni zvunec");
        lcd.setCursor(0, 1);
        lcd.print("Izberi: ");
        lcd.print(editingHour);
        lcd.print(":");
        Serial.println(editingMinute);

        if (editingMinute < 10) {
          lcd.print("0");
        };
        lcd.print(editingMinute);
      } else {
        lcd.setCursor(0, 0);
        lcd.print("Promeni zvuncec");
        lcd.setCursor(0, 1);
        lcd.print("Izberi: ");
        lcd.print("^");
      }
      switch (press) {
        case LEFT:
          {
            Serial.println("Left");
            (Намалява минутите на текущия звънец с 5 минути, ако е възможно)
            if (removedMinutes < 20) {
              editingMinute -= 5;
              removedMinutes += 5;
              addedMinutes = 0;
              if (editingMinute < 0) {
                if (editingHour > 0) {
                  editingHour--;

                  editingMinute = 55;
                } else {
                  editingMinute = 0;
                }
              }
            }
            while (getButton() == LEFT) ;
          }
          break;
        case RIGHT:
          {
            Serial.println("Right");
            (Увеличава минутите на текущия звънец с 5 минути, ако е възможно)
            if (removedMinutes > addedMinutes) {
              editingMinute += 5;
              addedMinutes += 5;
              if (editingMinute > 59) {
                editingHour++;
                editingMinute = 0;
              }
            }
            while (getButton() == RIGHT) ;
          } break;

        case UP:
          {
            Serial.println("Up");
            (Преминава към звънеца с по-голям индекс)
            lcd.clear();
            if (currHour < 21) {
              if (currHour < 19) {
                currHour += 3;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              } else if (currHour >= 19) {
                currHour += 2;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              }
              Serial.println(bellSchedule[currHour].hour);
              Serial.println(bellSchedule[currHour].minute);

            }

            while (getButton() == UP) ;
          }
          break;

        case DOWN:
          {
            Serial.println("Down");
            (Преминава към звънеца с по-малък индекс)
            lcd.clear();
            if (currHour > 1) {
              if (currHour > 19) {
                currHour -= 2;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              } else if (currHour <= 19) {
                currHour -= 3;
                editingHour = bellSchedule[currHour].hour;
                editingMinute = bellSchedule[currHour].minute;
                removedMinutes = 0;
                addedMinutes = 0;
              }

            }
            while (getButton() == DOWN) ;
          }
          break;

        case SELECT:
          {
            Serial.println("Zapazeno!");
            while (getButton() == SELECT) ;

            (Запазва промените в графика на звънеца и коригира следващите часове)
            if (setSchoolBell) {
              setSchoolBell = false;
              settings = false;
              lcd.clear();
              lcd.print("Zapazeno!");
              delay(2000);
              lcd.clear();

              bellSchedule[currHour].hour = editingHour;
              bellSchedule[currHour].minute = editingMinute;

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
              editingHour = bellSchedule[currHour].hour;
              editingMinute = bellSchedule[currHour].minute;
            }
          }
          break;
          delay(500); (Допълнително забавяне)

      }
    }
  }
if (!settings) {
  (Ако не сме в режим на настройки — показва текущото време)
  displayTime();

  (Ако е работен ден (понеделник-петък), проверява графика на звънеца)
  if (dayOfWeek != 0 || dayOfWeek != 6) {
    checkBellSchedule();
  }
}

(Проверява дали е натиснат бутонът за нулиране)
if (digitalRead(RESET_BUTTON) == HIGH) {
  (Връща графика на звънеца към фабричните настройки)
  for (int i = 0; i < 19; i++) {
    bellSchedule[i].hour = bellScheduleRes[i].hour;
    bellSchedule[i].minute = bellScheduleRes[i].minute;
  }
  (Изчиства екрана и показва съобщение за нулиране)
  lcd.clear();
  lcd.setCursor(0,0);

  lcd.print("Pochistvane na");

  lcd.setCursor(0, 1);
  lcd.print("nastroikite");
  
  delay(3000);
  lcd.clear();
}
void displayTime() {
  (Показва текущия час и минути на LCD)
  lcd.setCursor(0, 0);
  lcd.print("Chas: ");
  lcd.print(RTC.getHours());
  lcd.print(":");
  (Добавя водеща нула ако минутите са под 10)
  if (RTC.getMinutes() < 10) lcd.print("0");
  lcd.print(RTC.getMinutes());
}
void ringBell(int type) {
  (Звъни с различна дължина според типа на звънеца)
  if (type == 1) {
    (Дълъг звънец: включва релето за дълго време)
    digitalWrite(RELAY_PIN, LOW);

    delay(RING_DURATION_LONG);
    digitalWrite(RELAY_PIN, HIGH);
  } else if (type == 2) {
    (Кратък звънец: два кратки звънеца с пауза между тях)
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
  (Взима текущия час и минути от RTC)
  int currentHour = RTC.getHours();
  int currentMinute = RTC.getMinutes();

  (Обхожда целия график за звънци)
  for (int i = 0; i < bellCount; i++) {
    (Ако времето съвпада с настройките на звънеца)
    if (bellSchedule[i].hour == currentHour && bellSchedule[i].minute == currentMinute) {

      (Звъни според типа на звънеца)
      ringBell(bellSchedule[i].type);

      (Показва съобщение на LCD за изчакване)
      lcd.setCursor(0, 0);
      lcd.print("Izchakaite 1 min");
      lcd.setCursor(0, 1);
      lcd.print("predi korekcii");

      (Изчиства екрана и изчаква 1 минута, за да не звъни пак веднага)
      lcd.clear();
      delay(60000);
      break;
    }
  }
}

