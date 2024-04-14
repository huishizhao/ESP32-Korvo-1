## yaml for ESP32-Korvo-1

Put partitions_esp32.csv into folder of homeassistant\config\esphome\
````
substitutions:
  name: esp32-korvo-1
  friendly_name: esp32-korvo-1
  voice_assist_idle_phase_id: "1"
  voice_assist_listening_phase_id: "2"
  voice_assist_thinking_phase_id: "3"
  voice_assist_replying_phase_id: "4"
  voice_assist_not_ready_phase_id: "10"
  voice_assist_error_phase_id: "11"
  voice_assist_muted_phase_id: "12"
  micro_wake_word_model: okay_nabu
esphome:
  name: ${name}
  friendly_name: ${friendly_name}
  min_version: 2023.12.8
  platformio_options:
    board_build.flash_mode: dio
  project:
    name: esphome.voice-assistant
    version: "2.0"
  on_boot:
    - priority: -100
      then:
        - light.turn_on:
            id: led_ring
            blue: 0%
            red: 100%
            green: 0%
            effect: Fast Pulse
        - delay: 1s
        - wait_until:
            condition:
              wifi.connected:
        - light.turn_on:
            id: led_ring
            blue: 0%
            red: 100%
            green: 50%
            effect: Slow Pulse
        - wait_until: 
            condition:
              api.connected
        - lambda: id(init_in_progress) = false;
        - lambda: id(voice_assistant_phase) = ${voice_assist_idle_phase_id};
        - script.execute: reset_led
esp32:
  board: esp-wrover-kit
  flash_size: 16MB
  framework:
    type: esp-idf
    version: recommended
    sdkconfig_options:
      CONFIG_IDF_TARGET_ESP32: y
      CONFIG_ESPTOOLPY_FLASHMODE_QIO: y
      CONFIG_ESPTOOLPY_FLASHFREQ_80M: y
      CONFIG_ESPTOOLPY_FLASHSIZE_16MB: y
      CONFIG_PARTITION_TABLE_CUSTOM: y
      CONFIG_PARTITION_TABLE_CUSTOM_FILENAME: "default_16MB.csv" #"partitions_esp32.csv"
      CONFIG_PARTITION_TABLE_FILENAME: "default_16MB.csv" #"partitions_esp32.csv"
      CONFIG_PARTITION_TABLE_OFFSET: "0x8000"
      CONFIG_ESP32_DEFAULT_CPU_FREQ_240: y
      CONFIG_ESP32_SPIRAM_SUPPORT: y
      CONFIG_SPIRAM_SPEED_80M: y
      CONFIG_ESP_SYSTEM_PANIC_SILENT_REBOOT: y
      CONFIG_I2S_ENABLE_DEBUG_LOG: y
      CONFIG_MODEL_IN_SPIFFS: y
#psram:
#  mode: octal
#  speed: 80MHz
external_components:
  - source: github://rpatel3001/esphome@es8311
    components: [ es8311 ]
#  - source: github://pr#4861
#    components: [es8311]
#    refresh: 0s
  - source: github://rpatel3001/esphome@es7210
    components: [ es7210 ]
  - source: github://pr#5230
    components:
      - esp_adf

# Enable logging
logger:
  level: VERBOSE

# Enable Home Assistant API
api:
  encryption:
    key: "vRvf5APYhFeBjsFt8zzQ6xpuiZqn3oCAIbyVHCBawWM="

ota:
  password: "9522b9fe61f659e429743438edf3240e"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Esp32-Korvo-1 Fallback Hotspot"
    password: "vBJEmQ5iJHQx"

captive_portal:

i2c:
  - id: bus
    sda: GPIO19
    scl: GPIO32
    scan: true
#    frequency: 400kHz

es8311:
  address: 0x18

es7210:
  address: 0x40

output:
  - platform: gpio
    id: pa_ctrl
    pin:
      number: GPIO12
      ignore_strapping_warning: true
i2s_audio:
  - id: codec
    i2s_lrclk_pin: GPIO22 
    i2s_bclk_pin: GPIO25 
    i2s_mclk_pin:
       number: GPIO0
       allow_other_uses: true
       ignore_strapping_warning: true
  - id: mic_adc
    i2s_lrclk_pin: GPIO26 
    i2s_bclk_pin: GPIO27 
    i2s_mclk_pin:
       number: GPIO0
       allow_other_uses: true
       ignore_strapping_warning: true

speaker:
  - platform: i2s_audio
    id: external_speaker
    dac_type: external
    i2s_audio_id: codec
    i2s_dout_pin: GPIO13
    mode: mono

microphone:
  - platform: i2s_audio
    id: external_mic
    adc_type: external
    i2s_audio_id: mic_adc
    i2s_din_pin: GPIO36
    pdm: false


micro_wake_word:
  model: ${micro_wake_word_model}  #okay_nabu
  on_wake_word_detected:
    then:
      - voice_assistant.start:
          wake_word: !lambda return wake_word;
esp_adf:

voice_assistant:
  id: voice_asst
  microphone: external_mic
  speaker: external_speaker
  noise_suppression_level: 2
  auto_gain: 15dBFS
  volume_multiplier: 0.5
  vad_threshold: 3
  on_listening:
    - lambda: id(voice_assistant_phase) = ${voice_assist_listening_phase_id};
    - script.execute: reset_led
  on_stt_vad_end:
    - lambda: id(voice_assistant_phase) = ${voice_assist_thinking_phase_id};
    - script.execute: reset_led
  on_tts_start:
    - light.turn_on:
        id: led_ring
        blue: 0%
        red: 100%
        green: 100%
        brightness: 60%
        effect: Working
  on_stt_end: 
    - homeassistant.service:
        service: media_player.play_media
        data:
          entity_id: media_player.ke_ting
          media_content_id: !lambda return x;
          media_content_type: music
          announce: "true"

  on_tts_stream_start:
    - output.turn_on: pa_ctrl
    - delay: 100ms
    - lambda: id(voice_assistant_phase) = ${voice_assist_replying_phase_id};
    - script.execute: reset_led

  on_end:
    - wait_until:
        not:
          speaker.is_playing:
    - lambda: id(voice_assistant_phase) = ${voice_assist_idle_phase_id};
    - script.execute: reset_led
    - if:
        condition:
          and:
            - switch.is_off: mute
            - lambda: return id(wake_word_engine_location).state == "On device";
        then:
          - wait_until:
              not:
                voice_assistant.is_running:
          - micro_wake_word.start:

  on_error:
    - if:
        condition:
          lambda: return !id(init_in_progress);
        then:
          - lambda: id(voice_assistant_phase) = ${voice_assist_error_phase_id};
          - script.execute: reset_led
          - delay: 2s
          - if:
              condition:
                switch.is_off: mute
              then:
                - lambda: id(voice_assistant_phase) = ${voice_assist_idle_phase_id};
              else:
                - lambda: id(voice_assistant_phase) = ${voice_assist_muted_phase_id};
          - script.execute: reset_led

  on_client_connected:
    - if:
        condition:
          switch.is_off: mute
        then:
          - if:
              condition:
                lambda: return id(wake_word_engine_location).state == "In Home Assistant";
              then:
                - lambda: id(voice_asst).set_use_wake_word(true);
                - voice_assistant.start_continuous:
          - if:
              condition:
                lambda: return id(wake_word_engine_location).state == "On device";
              then:
                - micro_wake_word.start
          - lambda: id(voice_assistant_phase) = ${voice_assist_idle_phase_id};
        else:
          - lambda: id(voice_assistant_phase) = ${voice_assist_muted_phase_id};
    - lambda: id(init_in_progress) = false;
    - script.execute: reset_led

  on_client_disconnected:
    - if:
        condition:
          lambda: return id(wake_word_engine_location).state == "In Home Assistant";
        then:
          - lambda: id(voice_asst).set_use_wake_word(false);
          - voice_assistant.stop:
    - if:
        condition:
          lambda: return id(wake_word_engine_location).state == "On device";
        then:
          - micro_wake_word.stop
    - lambda: id(voice_assistant_phase) = ${voice_assist_not_ready_phase_id};
    - script.execute: reset_led


script:
  - id: reset_led
    then:
      - if:
          condition:
            lambda: return !id(init_in_progress);
          then:
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_listening_phase_id};
                then:                     
                  - light.turn_on:
                      id: led_ring
                      blue: 0%
                      red: 0%
                      green: 100%
                      brightness: 100%
                      effect: wakeword
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_thinking_phase_id};
                then:                     
                  - light.turn_on:
                      id: led_ring
                      blue: 100%
                      red: 100%
                      green: 0%
                      brightness: 100%
                      effect: Working
                  - delay: 100ms
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_replying_phase_id};
                then:                     
                  - light.turn_on:
                      id: led_ring
                      blue: 100%
                      red: 0%
                      green: 0%
                      brightness: 100%
                      effect: Working
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_idle_phase_id};
                then:
                  - light.turn_on:
                      id: led_ring
                      blue: 100%
                      red: 0%
                      green: 0%
                      brightness: 40%
                      effect: none
                  - delay: 200ms
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_not_ready_phase_id};
                then:                     
                  - light.turn_on:
                      id: led_ring
                      blue: 40%
                      red: 100%
                      green: 0%
                      effect: Slow Pulse
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_error_phase_id};
                then:                     
                  - light.turn_on:
                      id: led_ring
                      blue: 0%
                      red: 100%
                      green: 0%
                      brightness: 100%
                      effect: none
            - if:
                condition:
                  lambda: return id(voice_assistant_phase) == ${voice_assist_muted_phase_id};
                then:                     
                  - light.turn_off: led_ring
          else:
            - light.turn_on:
                id: led_ring
                blue: 0%
                red: 100%
                green: 0%
                effect: Fast Pulse

light:
  - platform: esp32_rmt_led_strip
    id: led_ring
    name: "${friendly_name} Light"
    pin: GPIO33 #GPIO19
    num_leds: 12
    rmt_channel: 0
    rgb_order: GRB
    chipset: ws2812
    default_transition_length: 0s
    effects:
      - pulse:
          name: "Pulse"
          transition_length: 300ms
          update_interval: 300ms
          min_brightness: 50%
          max_brightness: 100%

      - addressable_twinkle:
          name: "Working"
          twinkle_probability: 5%
          progress_interval: 3ms
      - addressable_color_wipe:
          name: "Wakeword"
          colors:
            - red: 0%
              green: 50%
              blue: 0%
              num_leds: 12
          add_led_interval: 40ms
          reverse: false
      - pulse:
          name: "Slow Pulse"
          transition_length: 0.5s
          update_interval: 1s
          min_brightness: 0%
          max_brightness: 100%
      - pulse:
          name: "Fast Pulse"
          transition_length: 50ms
          update_interval: 100ms
          min_brightness: 50%
          max_brightness: 100%

switch:
  - platform: template
    name: Mute
    id: mute
    optimistic: true
    restore_mode: RESTORE_DEFAULT_OFF
    entity_category: config
    on_turn_off:
      - if:
          condition:
            lambda: return !id(init_in_progress);
          then:
            - lambda: id(voice_assistant_phase) = ${voice_assist_idle_phase_id};
            - if:
                condition:
                  not:
                    - voice_assistant.is_running
                then:
                  - if:
                      condition:
                        lambda: return id(wake_word_engine_location).state == "In Home Assistant";
                      then:
                        - lambda: id(voice_asst).set_use_wake_word(true);
                        - voice_assistant.start_continuous
                  - if:
                      condition:
                        lambda: return id(wake_word_engine_location).state == "On device";
                      then:
                        - micro_wake_word.start
            - script.execute: reset_led
    on_turn_on:
      - if:
          condition:
            lambda: return !id(init_in_progress);
          then:
            - lambda: id(voice_asst).set_use_wake_word(false);
            - voice_assistant.stop
            - micro_wake_word.stop
            - lambda: id(voice_assistant_phase) = ${voice_assist_muted_phase_id};
            - script.execute: reset_led
  - platform: restart
    name: "${name} Restart"
select:
  - platform: template
    entity_category: config
    name: Wake word engine location
    id: wake_word_engine_location
    optimistic: true
    restore_value: true
    options:
      - In Home Assistant
      - On device
    initial_option: On device
    on_value:
      - wait_until:
          lambda: return id(voice_assistant_phase) == ${voice_assist_muted_phase_id} || id(voice_assistant_phase) == ${voice_assist_idle_phase_id};
      - if:
          condition:
            lambda: return x == "In Home Assistant";
          then:
            - micro_wake_word.stop
            - delay: 500ms
            - if:
                condition:
                  switch.is_off: mute
                then:
                  - lambda: id(voice_asst).set_use_wake_word(true);
                  - voice_assistant.start_continuous:
      - if:
          condition:
            lambda: return x == "On device";
          then:
            - lambda: id(voice_asst).set_use_wake_word(false);
            - voice_assistant.stop
            - delay: 500ms
            - micro_wake_word.start

globals:
  - id: init_in_progress
    type: bool
    restore_value: false
    initial_value: "true"
  - id: voice_assistant_phase
    type: int
    restore_value: false
    initial_value: ${voice_assist_not_ready_phase_id}

binary_sensor:
  - platform: template
    name: "${friendly_name} Volume Up"
    id: btn_volume_up
    publish_initial_state : True
  - platform: template
    name: "${friendly_name} Volume Down"
    id: btn_volume_down
    publish_initial_state : True
  - platform: template
    name: "${friendly_name} Set"
    id: btn_set
    publish_initial_state : True
  - platform: template
    name: "${friendly_name} Play"
    id: btn_play
    publish_initial_state : True
  - platform: template
    name: "${friendly_name} Mode"
    id: btn_mode
    publish_initial_state : True
  - platform: template
    name: "${friendly_name} Record"
    id: btn_record
    publish_initial_state : True
    on_press:
      - voice_assistant.start:
      - light.turn_on:
          id: led_ring
          blue: 0%
          red: 0%
          green: 100%
          brightness: 100%
          effect: "Wakeword"
#    on_release:
#      - voice_assistant.stop:
#      - output.turn_off: pa_ctrl
#      - light.turn_off:
#          id: led_ring
sensor:
  - id: button_adc
    platform: adc
    internal: true
    pin: 39 #8
    attenuation: 11db
    update_interval: 15ms
    filters:
      - median:
          window_size: 5
          send_every: 5
          send_first_at: 1
      - delta: 0.1
    on_value_range:
      - below: 0.55
        then:
          - binary_sensor.template.publish:
              id: btn_volume_up
              state: ON
      - above: 0.65
        below: 0.92
        then:
          - binary_sensor.template.publish:
              id: btn_volume_down
              state: ON
      - above: 1.02
        below: 1.33
        then:
          - binary_sensor.template.publish:
              id: btn_set
              state: ON
      - above: 1.43
        below: 1.77
        then:
          - binary_sensor.template.publish:
              id: btn_play
              state: ON
      - above: 1.87
        below: 2.15
        then:
          - binary_sensor.template.publish:
              id: btn_mode
              state: ON
      - above: 2.25
        below: 2.56
        then:
          - binary_sensor.template.publish:
              id: btn_record
              state: ON
      - above: 2.8
        then:
          - binary_sensor.template.publish:
              id: btn_volume_up
              state: OFF
          - binary_sensor.template.publish:
              id: btn_volume_down
              state: OFF
          - binary_sensor.template.publish:
              id: btn_set
              state: OFF
          - binary_sensor.template.publish:
              id: btn_play
              state: OFF
          - binary_sensor.template.publish:
              id: btn_mode
              state: OFF
          - binary_sensor.template.publish:
              id: btn_record
              state: OFF
````

