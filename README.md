**Background**
Our story begins in August 2022 when I posted on the Home Assistant forum ( https://community.home-assistant.io/t/heat-recovery-mvhr-integration-titon-beam-in-ireland-mechanical-ventilation-with-heat-recovery/454942/22 ) on whether anyone has an integration for Titon/Beam Mechanical Ventilation with Heat Recovery (MVHR).
I didn't have much response, other than one or two responses looking for similar. I managed to get a command line to the unit via RS485 and was able to understand some of the concepts using a manual from the vendor.

After much playing around with it via direct serial, I found the serial bus always crashing if there was a clash with data coming from the MVHR, so I decided in early 2025 to go with a D1 Mini and a  TTL (UART) to RS485 Module.
Its been a busy year and I have done very little with it... so here's take two!

Here we go! ...
