

## Part 3

**Change1**:

```verilog
module decode(xop, ir);
output reg `EXTOP xop;	// decoded 7-bit op
input `INST ir;		// instruction

always @(ir) begin
  case (ir `OP)
    `RTYPE:	case (ir `FUNCT)
		  `ADDU, `SUBU,
		  `AND, `OR, `XOR,
		  `MIN, `SLTU, `JR:	xop = `F(ir `FUNCT);   // <<<<<<<<<<THIS LINE>>>>>>>>>>
		  default:	xop = `TRAP;
		endcase
    `BEQ,
    `ADDIU, `SLTIU,
    `ANDI, `ORI, `XORI,
    `LUI, `LW, `SW:	xop = ir `OP;
    default:		xop = `TRAP;
  endcase
end
endmodule
```

Added JR to the RTYPE case.

**Change2**

```verilog
    // JR test
    r[20] = 80; // r[20] = m[20]
    `RPACK(m[18], 20, 0, 0, 0, `JR) // use r[20] to jump to m[20]
    `IPACK(m[19], `ADDIU, 5, 12, -1) // if didn't jump, R12 will be added to 11810
```

Add JR test to the testbench, if the jump is successful, R12 will not be added to 11810.

**Change 3**

```verilog
// JR Part
wire PCtoReg;
wire `ADDR NextPC;
```

Added two wires, PCtoReg is for the JR instruction, NextPC is the next PC value.

**Change 4**

```verilog
// JR Part
assign PCtoReg = ((ir `OP == `RTYPE) && (ir `FUNCT == `JR)); // if instruction is JR
assign NextPC = (PCtoReg ? r[ir `RS] : BranchZeroMux);  // next pc addr will be RS instead of BranchZeroMux
```

if the instruction is JR, PCtoReg will be 1, and the NextPC will be the value in RS instead of BranchZeroMux.


**Change 5**

```verilog
always @(posedge clk) begin
  if (reset) begin
    // reset
    pc <= 0;
    halt <= 0;
    r[0] <= 0;
  end else begin
    // normal operation
    if (!halt) begin
      if (ALUcontrol != `TRAP) begin
`ifdef	TRACE
        if (ir `OP) $display("%d: OP=%x RS=%d RT=%d IMM=%x", pc, ir `OP, ir `RS, ir `RT, ir `IMM);
        else $display("%d: OP=%x RS=%d RT=%d RD=%d SHAMT=%d FUNCT=%x", pc, ir `OP, ir `RS, ir `RT, ir `RD, ir `SHAMT, ir `FUNCT);
        if (MemWrite) $display("%d: m[%d] <= r[%d] (%d)", pc, ALUresult, ir `RT, r[ir `RT]);
        if (RegWrite) $display("%d: r[%d] <= %d", pc, RegDstMux, MemtoRegMux);

        // <<<<<<<<THIS LINE>>>>>>>>
        $display("%d: pc <= %d", pc, NextPC); // Display Next PC instead BranchZeroMux
`endif
        if (MemWrite) m[ALUresult >> 2] <= r[ir `RT];
        if (RegWrite) r[RegDstMux] <= MemtoRegMux;

        // <<<<<<<<THIS LINE>>>>>>>>
        pc <= NextPC; // The next PC address is determined by the JR instruction and BranchZeroMux
      end else halt <= 1;
    end
  end
end
endmodule
```

changed the display line to show NextPC instead of BranchZeroMux, because next PC addr is now determined by NextPC.

PC will be updated to NextPC.