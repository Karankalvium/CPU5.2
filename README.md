CHIP CPU {

    IN  inM[16],         // M value input  (M = contents of RAM[A])
        instruction[16], // Instruction for execution
        reset;           // Signals whether to re-start the current
                         // program (reset==1) or continue executing
                         // the current program (reset==0).

    OUT outM[16],        // M value output
        writeM,          // Write to M? 
        addressM[15],    // Address in data memory (of M)
        pc[15];          // address of next instruction

    PARTS:
   /** Instruction decoding:
    * There are three d-bits in the instruction, {A, D, M}, which determines which register is to accept the ALU output. **/
    DMux(in=true, sel=instruction[15], a=atype, b=ctype);
    Or(a=atype, b=instruction[5], out=ainstruct);
    And(a=ctype, b=instruction[4], out=cinstruct);

    /** A register **/
    Mux16(a=aluout, b=instruction, sel=atype, out=toareg);
    ARegister(in=toareg, load=ainstruct, out=aregout, out[0..14]=addressM);

    /** D register and the ALU **/
    Mux16(a=aregout, b=inM, sel=instruction[12], out=inputsel);

    ALU(x=dregout, y=inputsel, zx=instruction[11], nx=instruction[10], zy=instruction[9], ny=instruction[8], 
    f=instruction[7], no=instruction[6], out=aluout, zr=zrout, ng=ngout, out=outM);
    And(a=ctype, b=instruction[3], out=writeM);

    DRegister(in=aluout, load=cinstruct, out=dregout);

    /** Program Counter:
    * We need to choose between the operations PC=A and PC++ (the default). The jump condition determines if PC=A happens
    * or not. As we know from the Hack machine language there are six different conditions, with the exception of the
    * default JMP, they are JGT, JEQ, JGE, JLT, JNE, and JLE. However, we only need JGT and JLE to cover all cases.
    * To our aid we have the outputs 'zrout' and 'ngout' of the ALU, which determine if the output is zero or negative. 
    * The first three bits of the instruction specifies the jump condition, what each bit does is seen in Fig. 4.5 in 
    * The Elements of Computing Systems. **/
    Or(a=zrout, b=ngout, out=leqzero);		// the ALU tells us if out <=0
    Not(in=leqzero, out=posout);		// if not, out>0

    And(a=instruction[0], b=posout, out=jgt);	// if out>0 jump
    And(a=instruction[1], b=zrout, out=jeq);	// if out=0 jump
    And(a=instruction[2], b=ngout, out=jlt); 	// if out<0 jump
    Or(a=jeq, b=jlt, out=jle);			// if out<=0 jump
    Or(a=jgt, b=jle, out=jmp);			// covers all jump conditions

    And(a=jmp, b=ctype, out=dojump);		// the C instruction tells us to jump
    Not(in=dojump, out=nojump);			// or increment
    PC(in=aregout, load=dojump, inc=nojump, reset=reset, out[0..14]=pc);


}
