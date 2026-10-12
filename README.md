# FPGA-Tile

implementation code:

```verilog
module lut4 (

    input A,
    input B,
    input C,
    input D,
    
    input [15:0] LUT_config, //programmed_value

    output Y
);

wire [3:0] select;
assign select = {A, B, C, D};
assign Y = LUT_config[select];

endmodule

// The above code is equivalent to :

/*  
module lut (                      // look up table is equivalent to a mux, where the select lines of the mux are i/p of lut and i/ps of mux are the outputs in the truth table of lut. based on i/ps (select lines) there are programmed o/ps (i/ps of mux)
input a, b, c, d,
input [15:0]  programmed_value,
output reg y
);

always@(*) begin
case({a,b,c,d})
4'b0000: y = programmed_value[0];
4'b0001: y = programmed_value[1];
4'b0010: y = programmed_value[2];
4'b0011: y = programmed_value[3];
4'b0100: y = programmed_value[4];
4'b0101: y = programmed_value[5];
4'b0110: y = programmed_value[6];
4'b0111: y = programmed_value[7];
4'b1000: y = programmed_value[8];
4'b1001: y = programmed_value[9];
4'b1010: y = programmed_value[10];
4'b1011: y = programmed_value[11];
4'b1100: y = programmed_value[12];
4'b1101: y = programmed_value[13];
4'b1110: y = programmed_value[14];
4'b1111: y = programmed_value[15];
endcase
end

endmodule
*/


module d_flipflop (

    input clk,
    input rst,
    input d,

    output reg q
);

always @(posedge clk)
begin

    if(rst)
        q <= 1'b0;

    else
        q <= d;

end

endmodule



module mux2to1 (

    input a,
    input b,

    input sel,

    output y
);

assign y = (sel) ? b : a;

endmodule




module clb(
input  wire a, b, c, d,    //consider that ports diagram in textbook
input [15:0] lut_config,
input clk, rst,
input sel,
output wire  y
);

wire y1;
wire y2;

lut4 my_lut(.A(a), .B(b), .C(c), .D(d), .Y(y1), .LUT_config(lut_config) );

d_flipflop dff1(.clk(clk), .rst(rst), .d(y1), .q(y2)); 

mux2to1 my_mux(.a(y1) , .b(y2) ,.sel(sel), .y(y) ); 

endmodule


module mux4to1 (
    input  [3:0] in,
    input  [1:0] sel,
    output y
);

assign y = in[sel];

endmodule


// 4x4 Programmable Switch Matrix

module switch_matrix (
    
    input  [3:0] in,

    // 2-bit select for each output
    input  [1:0] sel0,
    input  [1:0] sel1,
    input  [1:0] sel2,
    input  [1:0] sel3,

    output [3:0] out
);

mux4to1 mux0 (
    .in(in),
    .sel(sel0),
    .y(out[0])
);

mux4to1 mux1 (
    .in(in),
    .sel(sel1),
    .y(out[1])
);

mux4to1 mux2 (
    .in(in),
    .sel(sel2),
    .y(out[2])
);

mux4to1 mux3 (
    .in(in),
    .sel(sel3),
    .y(out[3])
);

endmodule


module fpga_tile (

    // External inputs
    input  [3:0] tile_in,

    // LUT configuration
    input  [15:0] lut_config,

    // Switch matrix configuration
    input  [1:0] sel0,
    input  [1:0] sel1,
    input  [1:0] sel2,
    input  [1:0] sel3,

    // CLB control
    input clk,
    input rst,
    input sel,

    // External output
    output tile_out
);

wire [3:0] switch_out;


// 4x4 Switch Matrix

switch_matrix my_switch_matrix (

    .in(tile_in),

    .sel0(sel0),
    .sel1(sel1),
    .sel2(sel2),
    .sel3(sel3),

    .out(switch_out)
);


// CLB

clb my_clb (

    .a(switch_out[3]),
    .b(switch_out[2]),
    .c(switch_out[1]),
    .d(switch_out[0]),

    .lut_config(lut_config),

    .clk(clk),
    .rst(rst),

    .sel(sel),

    .y(tile_out)
);

endmodule
```

testbench:

```verilog
module fpga_tile_tb;

    
    // Inputs to FPGA tile
    

    reg [3:0]  tile_in;
    reg [15:0] lut_config;

    // Switch matrix configuration
    reg [1:0] sel0;
    reg [1:0] sel1;
    reg [1:0] sel2;
    reg [1:0] sel3;

    // CLB control
    reg clk;
    reg rst;
    reg sel;

    // FPGA tile output
    wire tile_out;


    
    // DUT - Device Under Test
    

    fpga_tile dut (

        .tile_in(tile_in),

        .lut_config(lut_config),

        .sel0(sel0),
        .sel1(sel1),
        .sel2(sel2),
        .sel3(sel3),

        .clk(clk),
        .rst(rst),
        .sel(sel),

        .tile_out(tile_out)
    );


    
    // Clock generation
    

    always #5 clk = ~clk;


    
    // Test sequence
    

    initial begin

        // Initial values
        clk = 0;
        rst = 1;
        sel = 0;

        tile_in = 4'b0000;

        // Initially:
        // O0 = I0
        // O1 = I1
        // O2 = I2
        // O3 = I3

        sel0 = 2'b00;
        sel1 = 2'b01;
        sel2 = 2'b10;
        sel3 = 2'b11;


        
        // 4-input XOR LUT configuration
        
        // Y = A ^ B ^ C ^ D
        
        // LUT_config[index] corresponds to ABCD
        

        lut_config = 16'b0110_1001_1001_0110;


        
        // Reset
        

        #10;
        rst = 0;


        
        // TEST 1
        // ABCD = 0000
        // XOR = 0
        

        tile_in = 4'b0000;

        #10;


        
        // TEST 2
        // ABCD = 0001
        // XOR = 1
        

        tile_in = 4'b0001;

        #10;


        
        // TEST 3
        // ABCD = 0011
        // XOR = 0
        

        tile_in = 4'b0011;

        #10;


        
        // TEST 4
        // ABCD = 0111
        // XOR = 1
        

        tile_in = 4'b0111;

        #10;


        
        // TEST 5
        // ABCD = 1111
        // XOR = 0
        

        tile_in = 4'b1111;

        #10;


        
        // TEST 6
        // Change switch matrix routing
        //
        // A = I2
        // B = I0
        // C = I3
        // D = I1
        
        
        sel0 = 2'b01;
        sel1 = 2'b11;
        sel2 = 2'b00;
        sel3 = 2'b10;
        
        tile_in = 4'b1010;
        
        #10;


        
        // TEST 7
        // Select registered output
        

        sel = 1'b1;

        tile_in = 4'b1101;

        #20;

        lut_config = 16'b1110_1000_1000_0000;
        
                
        // Reset
        

        #10;
        rst = 1;
        
        #10;
        rst = 0;


        
        // TEST 1
        // ABCD = 0000
        

        tile_in = 4'b0000;

        #10;


        // TEST 2
        // ABCD = 0001
        
        

        tile_in = 4'b0001;

        #10;


        
        // TEST 3
        // ABCD = 0011
       
        

        tile_in = 4'b0011;

        #10;


        
        // TEST 4
        // ABCD = 0111
        
        

        tile_in = 4'b0111;

        #10;


        
        // TEST 5
        // ABCD = 1111
        
        

        tile_in = 4'b1111;

        #10;


        // TEST 6
        // ABCD = 1010
    

        tile_in = 4'b1010;
        
        #10;


        // TEST 7
        // ABCD = 1101       

        tile_in = 4'b1101;

        #20;      

        // End simulation

        $finish;

    end

endmodule
```
