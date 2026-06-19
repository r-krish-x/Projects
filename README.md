_________________________Design_______________________________________

module ram(clk,rst,en,wr,addr,indata,outdata);
  input clk,rst,en,wr;
  input [7:0] indata;
  input [3:0] addr;
  output reg [7:0] outdata;
  reg [7:0] mem [15:0];
          
always@(posedge clk) begin
    
  if(!rst) begin
    outdata <= 0;
  end
    
  else begin
    
    if(en==1 && wr==1) begin
      mem[addr] <= indata;
    end
   
    else if(en==1 && wr==0) begin
      outdata <= mem[addr];
    end
    
  end

end
 
endmodule

________________________Testbench_____________________________________

`timescale 1ns / 1ns
 
`include "uvm_macros.svh"
import uvm_pkg::*;

// interface

// interface is for connecting DUT and TB

interface ram_if();
  
  logic clk;
  logic rst;
  logic en;
  logic wr;
  logic [3:0] addr;
  logic [7:0] indata;
  logic [7:0] outdata;
  
endinterface

// transaction class

class transaction extends uvm_sequence_item;
  
  rand bit [7:0] indata;
  rand bit [3:0] addr;
  rand bit en;
  rand bit wr;
  rand bit rst;
  bit [7:0] outdata;

// constraint for en, so that write and read operations can occur
  constraint c1 {en == 1;}
  
  constraint c2 {wr dist {0 := 4, 1 := 6};};
  
  function new(input string inst = "transaction");
    super.new(inst);
  endfunction
  
  `uvm_object_utils_begin(transaction)
  `uvm_field_int(indata, UVM_DEFAULT)
  `uvm_field_int(addr, UVM_DEFAULT)
  `uvm_field_int(en, UVM_DEFAULT)
  `uvm_field_int(wr, UVM_DEFAULT)
  `uvm_field_int(rst, UVM_DEFAULT)
  `uvm_field_int(outdata, UVM_DEFAULT)
  `uvm_object_utils_end
  
endclass

// sequence class

class generator extends uvm_sequence #(transaction);
  
  `uvm_object_utils(generator)
  
  transaction t;
  
  function new(input string inst = "GEN");
    super.new(inst);
  endfunction
  
  virtual task body();
    t = transaction::type_id::create("t");

// sequence with full input data range
    repeat(50)
      begin
        start_item(t);
        t.randomize() with {t.rst == 1;};
        finish_item(t);
        `uvm_info("GEN",$sformatf("Data send to Driver address : %0d , input data : %0d",t.addr,t.indata),UVM_NONE);
      end

// sequence with low input data range
    repeat(50)
      begin
        start_item(t);
        t.randomize() with {t.indata inside {[0:100]} && t.rst == 1;};
        finish_item(t);
        `uvm_info("GEN",$sformatf("Data send to Driver address : %0d , input data : %0d",t.addr,t.indata),UVM_NONE);
      end

// sequence with medium input data range
    repeat(50)
      begin
        start_item(t);
        t.randomize() with {t.indata inside {[101:200]} && t.rst == 1;};
        finish_item(t);
        `uvm_info("GEN",$sformatf("Data send to Driver address : %0d , input data : %0d",t.addr,t.indata),UVM_NONE);
      end

// sequence with high input data range
    repeat(50)
      begin
        start_item(t);
        t.randomize() with {t.indata inside {[200:255]} && t.rst == 1;};
        finish_item(t);
        `uvm_info("GEN",$sformatf("Data send to Driver address : %0d , input data : %0d",t.addr,t.indata),UVM_NONE);
      end
  endtask
  
endclass

// sequence2 class

class generator2 extends uvm_sequence #(transaction);
  
  `uvm_object_utils(generator2)
  
  transaction t;
  
  function new(input string inst = "GEN2");
    super.new(inst);
  endfunction
  
  virtual task body();
    t = transaction::type_id::create("t");

// sequence for checking reset operation
      begin
        start_item(t);
        t.randomize() with {t.rst == 0;};
        finish_item(t);
        `uvm_info("GEN2",$sformatf("Data send to Driver address : %0d , input data : %0d, reset : %0d",t.addr,t.indata,t.rst),UVM_NONE);
      end
  endtask
  
endclass

// sequencer class

// sequencer class is for starting the sequences

class sequencer extends uvm_sequencer #(transaction);

  `uvm_component_utils(sequencer) 

  function new(input string inst = "SEQR", uvm_component c);
    super.new(inst, c);
  endfunction
  
endclass

// driver class

// driver class is for driving the data towards the interface

class driver extends uvm_driver #(transaction);

  `uvm_component_utils(driver)
 
  function new(input string inst = "DRV", uvm_component c);
    super.new(inst, c);
  endfunction
 
  transaction data;
  virtual ram_if aif;

// task for reset operation
  task reset_dut();
    aif.rst <= 0;
    aif.addr <= 0;
    aif.indata <= 0;
    repeat(5) @(posedge aif.clk);
    aif.rst <= 1;
    #20;
    `uvm_info("DRV","Reset Done",UVM_NONE);
  endtask

  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    data = transaction::type_id::create("data");

// getting the interface parameter as set by configuration in top module
    if(!uvm_config_db #(virtual ram_if)::get(this,"","aif",aif)) 
    `uvm_error("DRV","Unable to access uvm_config_db");
  endfunction
 
  virtual task run_phase(uvm_phase phase);
    reset_dut();
    forever begin 
      seq_item_port.get_next_item(data);

// input data is driven only for write operations
      if(data.wr == 1) begin
        aif.indata <= data.indata;
      end
      aif.addr <= data.addr;
      aif.wr <= data.wr;
      aif.en <= data.en;
      aif.rst <= data.rst;
      seq_item_port.item_done(); 
      `uvm_info("DRV",$sformatf("Trigger DUT addr : %0d , indata : %0d",data.addr, data.indata),UVM_NONE); 
      @(posedge aif.clk);
    end
  endtask

endclass

// monitor class

// monitor class is for sampling the data received from interface

class monitor extends uvm_monitor;

  `uvm_component_utils(monitor)

  // analysis port of transaction type is for sending the data to the scoreboard
  uvm_analysis_port #(transaction) send;

  function new(input string inst = "MON", uvm_component c);
    super.new(inst, c);
    send = new("Write", this);
  endfunction
 
  transaction t;
  virtual ram_if aif;
 
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    t = transaction::type_id::create("TRANS");

// getting the interface parameter as set by configuration in top module
    if(!uvm_config_db #(virtual ram_if)::get(this,"","aif",aif)) 
    `uvm_error("MON","Unable to access uvm_config_db");
  endfunction

  virtual task run_phase(uvm_phase phase);
    @(negedge aif.clk);
    fork
      forever begin
        @(posedge aif.clk);

// providing appropriate delay so that data is sampled and sent as per requirements
        #1;
        t.outdata = aif.outdata;
        send.write(t);
      end
    join_none
    forever begin
      @(posedge aif.clk);
      t.addr = aif.addr;
      t.indata = aif.indata;
      t.wr = aif.wr;
      t.en = aif.en;
      t.rst = aif.rst;
      `uvm_info("MON",$sformatf("Data send to Scoreboard addr : %0d , indata : %0d , outdata : %0d", t.addr,t.indata,t.outdata),UVM_NONE);
     end
  endtask

endclass

// scoreboard class

// scoreboard class is for checking success or error of DUT

class scoreboard extends uvm_scoreboard;

  `uvm_component_utils(scoreboard)

  uvm_analysis_imp #(transaction,scoreboard) recv;
  transaction data;

// memory internal to scoreboard
  bit [7:0] localmem [15:0];
  bit [7:0] exp_data;
 
  function new(input string inst = "SCO", uvm_component c);
    super.new(inst, c);
    recv = new("Read", this);
  endfunction

  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    data = transaction::type_id::create("TRANS");
  endfunction

// write method for receiving the observed data
  virtual function void write(input transaction t);
    data = t;

// comparison logic for read operations
    if(data.rst == 1) begin

// saving data in memory for write operations
      if(data.wr == 1) begin
        localmem[data.addr] = data.indata;
      end
    
      else begin

// preparing expected data for comparison
        exp_data = localmem[data.addr];

// success of DUT due to observed data being the same as expected data
        if(data.outdata == exp_data) begin
          `uvm_info("SCO",$sformatf("DUT SUCCESS ||||||||||||||||| addr : %0d , indata : %0d , outdata : %0d |||||||||||||||||||||||||||||||||| outdata : %0d and exp_data : %0d is the same", data.addr,data.indata,data.outdata,data.outdata,exp_data),UVM_NONE);
        end

// error of DUT due to observed data being the same as expected data
        else begin
          `uvm_error("SCO",$sformatf("DUT ERROR.. ||||||||||||||||| addr : %0d , indata : %0d , outdata : %0d |||||||||||||||||||||||||||||||||| outdata : %0d and exp_data : %0d is NOT the same", data.addr,data.indata,data.outdata,data.outdata,exp_data));
        end
        
      end
      
    end

// comparison logic for reset operation
    else if(data.rst == 0) begin
      
      if(data.outdata == 0) begin
        `uvm_info("SCO",$sformatf("DUT RESET SUCCESS ||||||||||||||||| addr : %0d , indata : %0d , outdata : %0d |||||||||||||||||||||||||||||||||| outdata : %0d and exp_data : 0 is the same", data.addr,data.indata,data.outdata,data.outdata),UVM_NONE);
      end
      
      else begin
        `uvm_error("SCO",$sformatf("DUT RESET ERROR.. ||||||||||||||||| addr : %0d , indata : %0d , outdata : %0d |||||||||||||||||||||||||||||||||| outdata : %0d and exp_data : 0 is NOT the same", data.addr,data.indata,data.outdata,data.outdata));
      end
      
    end
  endfunction

endclass

// agent class

// agent class is for containing sequencer, driver and monitor

class agent extends uvm_agent;

  `uvm_component_utils(agent)

  function new(input string inst = "AGENT", uvm_component c);
    super.new(inst, c);
  endfunction
 
  monitor m;
  driver d;
  sequencer seqr;
 
 
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    m = monitor::type_id::create("MON",this);
    d = driver::type_id::create("DRV",this);
    seqr = sequencer::type_id::create("SEQR",this);
  endfunction

  virtual function void connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    
// connection between driver and sequencer
    d.seq_item_port.connect(seqr.seq_item_export);
  endfunction

endclass

// environment class

// environment class is for containing scoreboard and agent

class env extends uvm_env;

  `uvm_component_utils(env)

  function new(input string inst = "ENV", uvm_component c);
    super.new(inst, c);
  endfunction

  scoreboard s;
  agent a;

  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    s = scoreboard::type_id::create("SCO",this);
    a = agent::type_id::create("AGENT",this);
  endfunction

  virtual function void connect_phase(uvm_phase phase);
    super.connect_phase(phase);

// connection between monitor and scoreboard
    a.m.send.connect(s.recv);
  endfunction
 
endclass

// test class

// test class is for containing environment and sequences

class test extends uvm_test;

  `uvm_component_utils(test)

  function new(input string inst = "TEST", uvm_component c);
    super.new(inst, c);
  endfunction
 
  generator gen;
  generator2 gen2;
  env e;
 
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    gen = generator::type_id::create("GEN",this);
    gen2 = generator2::type_id::create("GEN2",this);
    e = env::type_id::create("ENV",this);
  endfunction

  virtual task run_phase(uvm_phase phase);
    phase.raise_objection(this);
    
// starting first sequence
    gen.start(e.a.seqr);
    #60;
    
// starting second sequence
// reset operation after fixed delay
    gen2.start(e.a.seqr);
    #60;
    
// starting both sequences concurrently
    fork
      gen.start(e.a.seqr);
      begin

// reset operation after random delay in the range of 10 to 3000
        #($urandom_range(10,3000));
        gen2.start(e.a.seqr);
      end
    join
    #60;
    phase.drop_objection(this);
  endtask

endclass

// top module testbench

// top module testbench is for running the test

module ram_tb();

  ram_if aif();
  
  initial begin
    aif.clk = 0;
    aif.rst = 0;
    aif.wr = 0;
    aif.en = 0;
  end  
  
  always #10 aif.clk = ~aif.clk;

  ram dut (.clk(aif.clk), .rst(aif.rst), .en(aif.en), .wr(aif.wr), .addr(aif.addr), .indata(aif.indata), .outdata(aif.outdata));
 
  initial begin
    $dumpfile("dump.vcd");
    $dumpvars;
  end
  
  initial begin

// setting the interface parameter by configuration
    uvm_config_db #(virtual ram_if)::set(null, "*", "aif", aif);
    run_test("test");
  end

endmodule
