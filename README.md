# Загрузка Orange Pi Zero 3W: microSD (U-Boot) + NVMe через JMS583 USB 3.0

Схема двойной загрузки для Orange Pi Zero 3W (Allwinner A733), где:

- **microSD** — только U-Boot + ядро
- **NVMe SSD** — корневая файловая система, подключена через адаптер JMS583 в USB 3.0 (Type-C, правый порт)

## Железо

| Компонент | Модель |
|-----------|-------|
| Плата | Orange Pi Zero 3W (Allwinner A733, 8 ядер, Debian 13.5) |
| Адаптер | JMicron JMS583 (VID:PID 152d:0583, Gen 2 to PCIe Gen3x2 Bridge) |
| NVMe #1 | WD Black SN750 1TB (WDS100T3X0C) — эталонная система |
| NVMe #2 | Silicon Power 256GB (SPCC M.2 PCIe SSD) — клон для тестов |
| microSD | Samsung 64GB EVO Plus A1 (ext4, label `opi_root`) |
| Питание | Type-C 5V/3A (левый порт) |

## Схема подключения

```
┌──────────────┐     ┌───────────┐
│  USB 3.0     │────▶│  JMS583   │─── NVMe SSD
│  (правый)    │     │  адаптер  │   (корень /)
└──────────────┘     └───────────┘
       ▲
┌──────┴──────┐
│  microSD    │─── U-Boot + dtb + ядро
│  (U-Boot)   │
└─────────────┘
```

**Важно:** Для прожорливых NVMe (WD Black 1TB, 6-8 Вт) требуется USB-хаб с внешним питанием. Silicon Power 256GB (~4-5 Вт) работает напрямую от USB 3.0 порта Zero 3W (0.9 А).

## Процесс загрузки

1. Boot ROM (Allwinner A733) → microSD
2. U-Boot читает `/boot/orangepiEnv.txt` → `rootdev=UUID=...`
3. Ядро загружается, находит NVMe через USB
4. Монтирует корень с NVMe → система запущена

## Настройка SD-карты

На SD-карте должен быть `/boot/orangepiEnv.txt` со следующим содержимым:

```ini
verbosity=1
bootlogo=true
overlay_prefix=sun60i-a733
fdtfile=allwinner/sun60i-a733-orangepi-zero3w.dtb
rootdev=UUID=<UUID_твоего_NVMe>
rootfstype=ext4
usbstoragequirks=0bda:9210:u
extraargs=rootdelay=10
```

Ключевая строка — `rootdev=UUID=<...>` — указывает на раздел NVMe с корневой ФС.

Чтобы узнать UUID раздела:
```bash
sudo blkid /dev/sda1
```

## Клонирование системы на новый NVMe

1. Подключить оба NVMe (источник + цель) к Pi5 через USB-хаб с доп. питанием
2. Источник (WD 1TB) → `/dev/sda`, цель (пустой) → `/dev/sdb`

```bash
# 1. Форматирование цели
sudo wipefs -a /dev/sdb
sudo badblocks -wsv -b 4096 /dev/sdb   # полная проверка (1.5-2 ч для 256GB)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L opi_root -m 0 /dev/sdb1

# 2. Клонирование
sudo mount /dev/sda1 /mnt/source
sudo mount /dev/sdb1 /mnt/target
sudo rsync -a --info=progress2 \
  --exclude='/proc/*' --exclude='/sys/*' --exclude='/dev/*' \
  --exclude='/tmp/*' --exclude='/run/*' --exclude='/mnt/*' \
  /mnt/source/ /mnt/target/

# 3. Обновление UUID в fstab клона
NEW_UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
sudo sed -i "s/UUID=[a-f0-9-]*/UUID=$NEW_UUID/" /mnt/target/etc/fstab

sudo umount /mnt/source /mnt/target
```

4. Обновить `rootdev=UUID=...` в `/boot/orangepiEnv.txt` на SD-карте на UUID нового NVMe

## Результаты тестов скорости (Zero 3W, USB 3.0)

| Носитель | Чтение | Запись |
|----------|--------|--------|
| Silicon Power 256GB + JMS583 | **661 MB/s** | 1.3 GB/s (SLC-кэш) |
| SD-карта Samsung 64GB EVO Plus A1 | 68.8 MB/s | 28.5 MB/s |

NVMe через JMS583 в 10 раз быстрее SD по чтению и в 45 раз по записи.

## Проблемы и решения

| Проблема | Причина | Решение |
|----------|---------|---------|
| Чёрный экран после U-Boot | UUID в orangepiEnv.txt не совпадает с NVMe | Поправить `rootdev=UUID=...` на SD-карте |
| Диск отваливается при загрузке | Нехватка питания USB (WD Black 1TB) | USB-хаб с внешним питанием, либо более экономичный NVMe |
| `usbstoragequirks` | Параметр для RTL9210 (0bda:9210:u), не мешает JMS583 | Можно оставить как есть |

## Файлы SD-карты

```
/boot/
  orangepiEnv.txt       — конфиг U-Boot (rootdev=UUID=...)
  orangepiEnv.txt.nvme  — резервная копия для NVMe
  orangepiEnv.txt.sd    — резервная копия для SD-only загрузки
  boot.cmd / boot.scr   — скрипт U-Boot
```

## Даты и версии

- Настройка: 14.06.2026
- Debian 13.5 (Trixie), ядро 6.6.98-sun60iw2
- Прошивка JMS583: 2.14
