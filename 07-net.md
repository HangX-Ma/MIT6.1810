# Lab7: Networking

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

In this lab you will write an xv6 device driver for a network interface card (NIC).

## Your Job(hard)

I'm not familiar the network protocols, so I don't know the details of this e1000 device communication method. The hints almost guide you to finish this lab! My further study will be relative to network.

### _kernel/e1000_dev.h_

The command chosen work confuse me at first. But I only find two TX commands in this definition file, with which you can pass the test.

```c
/* Transmit Descriptor command definitions [E1000 3.3.3.1] */
#define E1000_TXD_CMD_EOP    0x01 /* End of Packet */
#define E1000_TXD_CMD_RS     0x08 /* Report Status */
```

### _kernel/e1000.c_

I guess `while` needs to be used in `e1000_recv` but not fully confirm. Because the hint says that _Then check if a new packet is available by checking for the E1000_RXD_STAT_DD bit in the status portion of the descriptor. If not, stop._. In addition, the `e1000_recv` return nothing.

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&e1000_lock);
  uint32 tdt = regs[E1000_TDT];
  /* previous transmission hasn't completed */
  if ((E1000_TXD_STAT_DD & tx_ring[tdt].status) == 0) {
    release(&e1000_lock);
    return -1;
  }

  if (tx_mbufs[tdt]) {
    mbuffree(tx_mbufs[tdt]);
  }

  tx_ring[tdt].addr = (uint64)m->head;
  tx_ring[tdt].length = m->len;
  // Only find these two definitions
  tx_ring[tdt].cmd = E1000_TXD_CMD_EOP | E1000_TXD_CMD_RS;
  tx_mbufs[tdt] = m;

  regs[E1000_TDT] = (tdt + 1) % TX_RING_SIZE;
  release(&e1000_lock);

  return 0;
}

static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  while (1) {
    uint32 rdt = (regs[E1000_RDT] + 1) % RX_RING_SIZE;

    if ((E1000_RXD_STAT_DD & rx_ring[rdt].status) == 0) {
      break;
    }

    rx_mbufs[rdt]->len = rx_ring[rdt].length;
    net_rx(rx_mbufs[rdt]);

    rx_mbufs[rdt] = mbufalloc(0);
    rx_ring[rdt].addr = (uint64)rx_mbufs[rdt]->head;
    rx_ring[rdt].status = 0;

    regs[E1000_RDT] = rdt;
  }
}
```
