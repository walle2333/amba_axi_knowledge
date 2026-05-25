# Signal Table

## Global

| Signal | 含义 |
| --- | --- |
| `ACLK` | AXI clock |
| `ARESETn` | active-low reset |

## Write Address Channel

| Signal | 含义 |
| --- | --- |
| `AWID` | write transaction ID |
| `AWADDR` | write address |
| `AWLEN` | burst length minus 1 |
| `AWSIZE` | bytes per beat encoding |
| `AWBURST` | burst type |
| `AWCACHE` | cache/buffer attribute |
| `AWPROT` | protection/security attribute |
| `AWQOS` | QoS hint |
| `AWVALID/AWREADY` | handshake |

## Write Data Channel

| Signal | 含义 |
| --- | --- |
| `WDATA` | write data |
| `WSTRB` | byte lane strobe |
| `WLAST` | last write beat |
| `WVALID/WREADY` | handshake |

## Write Response Channel

| Signal | 含义 |
| --- | --- |
| `BID` | write response ID |
| `BRESP` | write response |
| `BVALID/BREADY` | handshake |

## Read Address Channel

| Signal | 含义 |
| --- | --- |
| `ARID` | read transaction ID |
| `ARADDR` | read address |
| `ARLEN` | burst length minus 1 |
| `ARSIZE` | bytes per beat encoding |
| `ARBURST` | burst type |
| `ARCACHE` | cache/buffer attribute |
| `ARPROT` | protection/security attribute |
| `ARQOS` | QoS hint |
| `ARVALID/ARREADY` | handshake |

## Read Data Channel

| Signal | 含义 |
| --- | --- |
| `RID` | read response ID |
| `RDATA` | read data |
| `RRESP` | read response |
| `RLAST` | last read beat |
| `RVALID/RREADY` | handshake |
