# Yandex-sound-TTS
**Описание:**
Скрипт предназначен для настройки громкости уведомлений и возвращение к предыдущему статусу, а также состояния колонки, если играла музыка, то колонка выставляет паузу, говорит с нужной громкостью из yandex_volume_report и возвращает громкость обратно, а также возобновляет музыку, если была включена.
Для ночного, дневного режима управляйте громкостью уведомлений yandex_volume_report.

**Как использовать:**
Для запуска голосового сообщения нужно запустить скрипт (см. ниже пример) и объявить переменные для настройки local, volume, station, reporter

**station**: сущность вашей колонки

**reporter**: Ваше сообщение и/или спецэффекты в режиме медиа

**local**: 0 для текста, 1 для медиа, 2 командный режим
По умолчанию 0

**volume**: 0.1 громкость TTS
По умолчанию input_number.yandex_volume_report

Для постоянной громкости, например, для протечки вы можете в скрипте поставить переменную volume: 1.0 (100%) и тогда эта громкость будет использована для этого скрипта всегда и будет, в этом случае, максимальной. 

**Требования:**
Для работы требуется настройка Яндекс диалогов с доступом по https
Все эффекты голоса и звуки работают только в режиме media через яндекс диалоги.
А также измените имя **media_content_type: text:кот Саймон**
кот Саймон на название вашего диалога

**Пример:**
_Пример также можно подсмотреть через шаблоны служб в панели разработчика _
```
script: #Для проверки, если не установлена громкость, то берет ее из   input_number yandex_volume_report:
   coocoo:
      alias: Кукушка скажи # Устанавливаем громкость на колонках из слайдера TTS Уведомлений
      icon: mdi:progress-wrench
      sequence:
      - service: script.turn_on
        target:
          entity_id: script.tts_yandex_sound
        data:
          variables:
            station: media_player.yandex_box_3 #media_player.yandex_box_3
            reporter: Кукушка сказала <speaker audio="alice-sounds-animals-cuckoo-1.opus">
            local: 1
            #volume: 0.9
```
# для Package
```
tts_yandex:

  input_number:
    yandex_volume_report:
      name: "Громкость TTS уведомлений"
      icon: mdi:volume-vibrate
      min: 0
      max: 100
      step: 10

  script:
    tts_yandex_sound:
      alias: TTS система уведомлений
      mode: parallel
      max: 10
      icon: mdi:bullhorn
      fields:
        station:
          description: ID Яндекс станции
          required: false #true
          example: media_player.yandex_box_3
          default: media_player.yandex_box_3
        reporter:
          description: Текст сообщения
          required: false #true
          example: >-
            <speaker audio="alice-sounds-things-old-phone-1.opus">
            <speaker effect="megaphone"> Кто-то звонит в дверь
        volume:
          description: Громкость
          required: false #true
          example: 0.9
          default: '{{ (states("input_number.yandex_volume_report")|int(default=50) /100 )|round(1) }}'
        local:
          description: 0-текст, 1-медиа, 3-TTS, 2-команда
          required: false
          example: 1
          default: 0
      sequence:
        - alias: Запоминаем состояние Яндекс станции
          variables:
            volume:   '{{volume|default((states("input_number.yandex_volume_report")|int(default=50) /100 )|round(1))|float}}'
            station:  '{{station|default("media_player.yandex_box_3")}}'
            local:    '{{local|default(0)}}'
            reporter: '{{reporter|default("Не указана переменная реп+ортэрр в скр+ипте")}}'
            old_state:  '{{ states(station) }}'
            old_volume: '{{ state_attr(station, "volume_level")|float(0) }}'

        - alias: Ставим музыку на паузу, если играет
          choose:
            conditions:
              - condition: template
                value_template: '{{ old_state == "playing" }}'
            sequence:
              - service: media_player.media_pause
                target:
                  entity_id: '{{ station }}'
                  
        - alias: Установить громкость колонки для 1,2,3 режима
          choose:
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) != 0 }}'
              sequence:
                - service: media_player.volume_set
                  target:
                    entity_id: '{{ station }}'
                  data:
                    volume_level: '{{ volume }}'
        - alias: Воспроизводим способ оповещения
          choose:
            #0 Способ TTS со встроенной настройкой звука
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) == 0 }}'
              sequence:
                - alias: Запускаем произношение текста (по-умолчанию)
                  service: tts.yandex_station_say
                  target:
                    entity_id: '{{ station }}'
                  data:
                    message: '{{ reporter }}'
                    options:
                      volume_level: '{{ volume }}'
            #1 Способ Медиа через диалог
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) == 1 }}'
              sequence:
                - alias: Запускаем произношение media
                  service: media_player.play_media
                  target:
                    entity_id: '{{ station }}'
                  data:
                    media_content_type: text:кот Саймон
                    media_content_id: '{{ reporter }}'
            #2 Способ TTS с внешней настройкой звука
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) == 2 }}'
              sequence:
                - alias: Запускаем произношение media
                  service: tts.yandex_station_say
                  target:
                    entity_id: '{{ station }}'
                  data:
                    message: '{{ reporter }}'
            #3 Способ медиа командой
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) == 3 }}'
              sequence:
                - alias: Запускаем произношение media командой
                  service: media_player.play_media
                  target:
                    entity_id: '{{ station }}'
                  data:
                    media_content_type: command
                    media_content_id: '{{ reporter }}'
                    
        - alias: Ждём, пока начнёт говорить
          wait_template: '{{ state_attr(station, "alice_state") == "SPEAKING"}}'
          timeout: '00:00:10'
          continue_on_timeout: true
        - delay:
            milliseconds: 20
            #miliseconds: 200
        - alias: Ждём, пока начнёт говорить
          wait_template: '{{ state_attr(station, "alice_state") == "SPEAKING"}}'
          timeout: '00:00:10'
          continue_on_timeout: true
        - alias: Ждём, пока перестанет говорить
          wait_template: '{{ state_attr(station, "alice_state") == "IDLE"}}'
          timeout: '00:01:00'
          continue_on_timeout: true
        - alias: Возвращаем громкость для 1 и 2 режима
          choose:
            - conditions:
                - condition: template
                  value_template: '{{ local|default(0) != 0 }}'
              sequence:
                - service: media_player.volume_set
                  data:
                    volume_level: '{{ old_volume }}'
                  target:
                    entity_id: '{{ station }}'
        - alias: Снова запускаем музыку, если она играла, иначе возвращаем громкость
          choose:
            - conditions:
                - condition: template
                  value_template: '{{ old_state == "playing"}}'
              sequence:
                - service: media_player.media_play_pause
                  target:
                    entity_id: '{{ station }}'
```              
