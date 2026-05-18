script_content = r"""@echo off
chcp 65001 >nul
title Настройка сетевого адаптера
echo ==========================================
echo  Проверка и настройка сетевого адаптера
echo ==========================================
echo.

:: Ищем первый физический сетевой адаптер (исключаем виртуальные и Bluetooth)
echo [*] Поиск сетевого адаптера...

set "AdapterName="
set "AdapterInterface="

:: Получаем имя интерфейса через netsh
for /f "skip=2 tokens=4,* delims= " %%a in ('netsh interface show interface ^| findstr /I /V "Loopback Bluetooth Virtual Hyper-V VMware VirtualBox" ^| findstr "Подключен Отключен Connected Disconnected"') do (
    if not defined AdapterInterface (
        set "AdapterInterface=%%b"
        echo [+] Найден адаптер: %%b
    )
)

if not defined AdapterInterface (
    echo [-] Сетевой адаптер не найден!
    echo     Убедитесь, что сетевая карта установлена.
    pause
    exit /b 1
)

echo.
echo [*] Проверка состояния адаптера "%AdapterInterface%"...

:: Проверяем статус адаптера
for /f "tokens=3 delims=: " %%a in ('netsh interface show interface name="%AdapterInterface%" ^| findstr /I "Состояние Admin State"') do (
    set "AdapterState=%%a"
)

:: Если адаптер отключён — включаем
if /I "%AdapterState%"=="Отключен" (
    echo [!] Адаптер отключён. Включаем...
    netsh interface set interface name="%AdapterInterface%" admin=enabled
    if errorlevel 1 (
        echo [-] Ошибка: не удалось включить адаптер.
        echo     Запустите скрипт от имени Администратора.
        pause
        exit /b 1
    )
    echo [+] Адаптер успешно включён.
    :: Ждём инициализации
    timeout /t 3 /nobreak >nul
) else if /I "%AdapterState%"=="Disabled" (
    echo [!] Адаптер отключён. Включаем...
    netsh interface set interface name="%AdapterInterface%" admin=enabled
    if errorlevel 1 (
        echo [-] Ошибка: не удалось включить адаптер.
        echo     Запустите скрипт от имени Администратора.
        pause
        exit /b 1
    )
    echo [+] Адаптер успешно включён.
    timeout /t 3 /nobreak >nul
) else (
    echo [+] Адаптер уже активен.
)

echo.
echo ==========================================
echo  Настройка статического IP-адреса
echo ==========================================
echo [*] Применяем параметры:
echo     IP-адрес:  10.10.10.15
echo     Маска:     255.255.255.0
echo     Шлюз:      10.10.10.2
echo.

:: Устанавливаем статический IP
netsh interface ip set address name="%AdapterInterface%" static 10.10.10.15 255.255.255.0 10.10.10.2 1

if errorlevel 1 (
    echo [-] Ошибка при установке IP-адреса!
    echo     Возможные причины:
    echo     - Недостаточно прав (запустите от Администратора)
    echo     - Адаптер не готов
echo.
    pause
    exit /b 1
)

echo [+] Статический IP успешно установлен!
echo.

:: Очищаем DNS-кэш для применения настроек
echo [*] Очистка DNS-кэша...
ipconfig /flushdns >nul
echo [+] DNS-кэш очищен.

echo.
echo ==========================================
echo  Настройка завершена успешно!
echo ==========================================
echo.
echo Текущая конфигурация:
netsh interface ip show config name="%AdapterInterface%"
echo.
pause
"""

# Записываем файл
output_path = "/mnt/agents/output/setup_network.bat"
with open(output_path, "w", encoding="utf-8") as f:
    f.write(script_content)

print(f"Файл создан: {output_path}")
print(f"Размер: {len(script_content)} байт")
