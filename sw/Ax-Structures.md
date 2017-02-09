You can use the Python API to generate an initialised C structure,
which can then be passed to the C API at runtime. This is useful for
microcontrollers where Python is not available or appropriate, but the
full functionality of the C API is not required either.

An example of this process is given in
`[ax_structures.py](ax_structures.py)`. It opens a file descriptor for
source and header, writes c niceties, and pretty-prints the structures.

```python
radio = AxRadio()
write_ax_modulation_struct(f_c, f_h, "ax_mod_default", radio)
```

This creates an initialised structure of type `ax_modulation`. A
pointer to this can then be passed to C API functions such as
`ax_tx_on`, `ax_tx_packet` and so on. Here's a full example.

```c
#include "ax/ax.h"
#include "ax/ax_modulations_autogen.h"

void main()
{
  uint32_t initial_frequency = 4346e5;
  ax_config config;

  /* -------- config struct -------- */
  memset(&config, 0, sizeof(ax_config));

  config.clock_source = AX_CLOCK_SOURCE_TCXO;
  config.f_xtal = 16369000;

  config.synthesiser.A.frequency = initial_frequency;
  config.synthesiser.B.frequency = initial_frequency;
  if (initial_frequency < 400e6) { /* external inductor used to tune VCO */
    config.synthesiser.vco_type = AX_VCO_INTERNAL_EXTERNAL_INDUCTOR;
  }

  /* TODO initialise spi hardware, and write spi_transfer wrapper */
  config.spi_transfer = ax_rf_spi_transfer;

  config.pkt_store_flags = AX_PKT_STORE_RSSI |
    AX_PKT_STORE_RF_OFFSET;

  /* ------- init ------- */
  ax_init_status result = ax_init(&config);

  if (result != AX_INIT_OK) {
    while (1);                  /* radio failed to initialise */
  }

  /* transmit */
  ax_tx_on(&config, &ax_mod_default); // ax_mod_default autogenerated by python API
  ax_tx_packet(&config, &ax_mod_default, "hello", 5);
  ax_off(&config);

  /* receive */
  ax_packet pkt;
  int rx_packet_result;

  ax_rx_on(&config, &ax_mod_default); // ax_mod_default autogenerated by python API
  do {
    rx_packet_result = ax_rx_packet(&config, &pkt);
  } while (!rx_packet_result);
  ax_off(&config);
}
```