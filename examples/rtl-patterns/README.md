# RTL Patterns

用于保存协议相关 RTL 模式和伪代码。

## Planned Examples

- VALID/READY source rule。
- VALID/READY sink rule。
- Skid buffer。
- Write channel join。
- Burst address generator。
- Outstanding counter。

## Source-Side Handshake Pattern

```verilog
always_ff @(posedge clk) begin
    if (!rst_n) begin
        valid <= 1'b0;
    end else if (!valid || ready) begin
        valid <= have_payload;
        data  <= next_payload;
    end
end
```

含义：当前 payload 未有效，或已经被接收时，才允许更新 `valid/data`。
