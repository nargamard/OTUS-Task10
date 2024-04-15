#Домашнее задание: работаем с процессами

Задания на выбор:

написать свою реализацию ps ax используя анализ /proc
Результат ДЗ - рабочий скрипт который можно запустить
написать свою реализацию lsof
Результат ДЗ - рабочий скрипт который можно запустить
дописать обработчики сигналов в прилагаемом скрипте, оттестировать, приложить сам скрипт, инструкции по использованию
Результат ДЗ - рабочий скрипт который можно запустить + инструкция по использованию и лог консоли
реализовать 2 конкурирующих процесса по IO. пробовать запустить с разными ionice
Результат ДЗ - скрипт запускающий 2 процесса с разными ionice, замеряющий время выполнения и лог консоли
реализовать 2 конкурирующих процесса по CPU. пробовать запустить с разными nice
Результат ДЗ - скрипт запускающий 2 процесса с разными nice и замеряющий время выполнения и лог консоли

---
## Решение:

Решил объединить два скрипта в один - сделать 2 конкурирующих процесса и по CPU, и по IO. Параметры для запуска скрипт берёт из файла `script_nice_ionice.conf`.

Текст скрипта:
```bash
#!/bin/bash

# Прочитаем переменные из файла конфигурации
source script_nice_ionice.conf

# Получаем имена временных файлов, но не создём их. Далее с такими именами сделаем именованные каналы.
FIFO_ETALON=$(mktemp -u)
FIFO_ETALON_CPU=$(mktemp -u)
FIFO_TEST=$(mktemp -u)
FIFO_TEST_CPU=$(mktemp -u)


# Создаём файлы именованных каналов
mkfifo "$FIFO_ETALON" "$FIFO_TEST" "$FIFO_ETALON_CPU" "$FIFO_TEST_CPU"

#Создадим тестовые файлы
FILE01="$TEST_DIR/FILE01"
FILE02="$TEST_DIR/FILE02"

#Включим ловушку, чтобы удалить временные файлы если скрипт завершится с ошибкой
trap 'rm -f "$FIFO_ETALON" "$FIFO_TEST" "$FIFO_ETALON_CPU" "$FIFO_TEST_CPU" "$FILE01" "$FILE02"' INT TERM EXIT

( (LANG=en_EN dd if=/dev/urandom of="$FILE01" oflag=direct bs="$BS" count="$COUNT" 2>&1) | tee "$FIFO_ETALON" > /dev/null)& PID_FIFO_ETALON=$!
( (LANG=en_EN nice -n"$NICE" ionice -c"$IONICECLASS" -n"$IONICE" dd if=/dev/urandom of="$FILE02" oflag=direct bs="$BS" count="$COUNT" 2>&1) | tee "$FIFO_TEST" > /dev/null)& PID_FIFO_TEST=$!


# Сообщим параметры запуска скрипта
echo "Эталонный процесс запущен без указания параметров nice и ionice."
echo "Второй процесс запущен с параметрами nice: -n $NICE и ionice -c$IONICECLASS -n$IONICE"

# Сохраним результаты работы процессов в переменные
RESULT_ETALON=$(grep "copied" "$FIFO_ETALON")
RESULT_TEST=$(grep "copied" "$FIFO_TEST")

# Ожидаем заверешение запущенных процессов
wait $PID_FIFO_ETALON $PID_FIFO_TEST

( (LANG=en_EN time dd if=/dev/urandom bs="$BS" count="$COUNT" | md5sum) 2>&1 | tee "$FIFO_ETALON_CPU" > /dev/null)& PID_FIFO_ETALON_CPU=$!
( (LANG=en_EN time nice -n"$NICE" ionice -c"$IONICECLASS" -n"$IONICE" dd if=/dev/urandom bs="$BS" count="$COUNT" | md5sum) 2>&1 | tee "$FIFO_TEST_CPU" > /dev/null)& PID_FIFO_TEST_CPU=$!

# Сохраним результаты работы процессов в переменные
RESULT_ETALON_CPU=$(grep "elapsed" "$FIFO_ETALON_CPU")
RESULT_TEST_CPU=$(grep "elapsed" "$FIFO_TEST_CPU")

# Ожидаем заверешение запущенных процессов
wait $PID_FIFO_ETALON_CPU $PID_FIFO_TEST_CPU

# Сообщим результат работы скрипта
echo "Эталонный результат: ${RESULT_ETALON#*copied, }"
echo "Тестовый результат: ${RESULT_TEST#*copied, }"
echo "Эталонный результат ЦПУ: ${RESULT_ETALON_CPU#*elapsed, }"
echo "Тестовый результат ЦПУ: ${RESULT_TEST_CPU#elapsed, }"

# Удалим временные файлы
rm -f "$FIFO_ETALON" "$FIFO_TEST" "$FIFO_ETALON_CPU" "$FIFO_TEST_CPU" "$FILE01" "$FILE02"
```