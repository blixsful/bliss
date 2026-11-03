---
title: "SmallSol"
---

Panduan untuk membuat machine emulator kecil menggunakan go.

Requirements:
- Pengetahuan dasar tentang golang, dan pengetahuan lebih lanjut dalam pemrograman.
- Host OS Linux / WSL (opsional, disarankan)
- Golang terinstall
- riscv64-unknown-elf-gcc terinstall

Panduan ini akan memandu proses pembuatan emulasi mesin untuk RISC-V RV32i.

## Konsep Dasar & Permulaan
CPU RV32I hanya butuh beberapa komponen:
• 32 register 32-bit (x0–x31). x0 selalu bernilai 0.
• Program counter (PC), alamat instruksi berikutnya.
• Memori berupa array byte, little-endian.
• Siklus fetch → decode → execute, diulang terus sampai program berhenti.

Setiap instruksi RV32I berukuran tepat 32 bit dan punya 6 format (R, I, S, B, U, J). Perbedaan format hanya pada cara bit-bit immediate disusun.

### Struktur Proyek

Struktur proyek akan terlihat seperti yang dibawah ini:
```SmallSol/
--go.mod
--decode.go
--cpu.go
--main.go
--test.go  // File untuk tes
```

Buat proyek:
```bash
mkdir SmallSol && cd SmallSol
go mod init SmallSol
```

## Decode

Decode itu memecah instruksi ya.
Ide utamanya: bungkus uint32 dalam tipe Inst lalu beri method untuk tiap field.

```
package main

// Opcode utama RV32I.
const (
	opLoad   = 0x03
	opFence  = 0x0F
	opImm    = 0x13
	opAUIPC  = 0x17
	opStore  = 0x23
	opReg    = 0x33
	opLUI    = 0x37
	opBranch = 0x63
	opJALR   = 0x67
	opJAL    = 0x6F
	opSystem = 0x73
)

type Inst uint32

func (i Inst) Opcode() uint32 { return uint32(i) & 0x7f }
func (i Inst) Rd() uint32     { return uint32(i>>7) & 0x1f }
func (i Inst) Funct3() uint32 { return uint32(i>>12) & 0x7 }
func (i Inst) Rs1() uint32    { return uint32(i>>15) & 0x1f }
func (i Inst) Rs2() uint32    { return uint32(i>>20) & 0x1f }
func (i Inst) Funct7() uint32 { return uint32(i >> 25) }

// Immediate. Konversi ke int32 lalu geser kanan = sign extension otomatis.

func (i Inst) ImmI() int32 { return int32(i) >> 20 }

func (i Inst) ImmS() int32 {
	return int32(i)>>25<<5 | int32(i>>7)&0x1f
}

func (i Inst) ImmB() int32 {
	return int32(i)>>31<<12 |
		int32(i>>7)&1<<11 |
		int32(i>>25)&0x3f<<5 |
		int32(i>>8)&0xf<<1
}

func (i Inst) ImmU() uint32 { return uint32(i) & 0xfffff000 }

func (i Inst) ImmJ() int32 {
	return int32(i)>>31<<20 |
		int32(i)&0xff000 |
		int32(i>>20)&1<<11 |
		int32(i>>21)&0x3ff<<1
}
```

## CPU dan Memori (cpu.go)
### Bagian 1
```
package main

import (
	"encoding/binary"
	"fmt"
	"os"
)

type CPU struct {
	Reg      [32]uint32
	PC       uint32
	Mem      []byte
	Halted   bool
	ExitCode int
}

func NewCPU(memSize int) *CPU {
	c := &CPU{Mem: make([]byte, memSize)}
	c.Reg[2] = uint32(memSize) // sp di ujung memori
	return c
}

func (c *CPU) setReg(rd, v uint32) {
	if rd != 0 { // x0 hardwired ke 0
		c.Reg[rd] = v
	}
}

func (c *CPU) check(addr, size uint32) error {
	if uint64(addr)+uint64(size) > uint64(len(c.Mem)) {
		return fmt.Errorf("akses memori di luar batas: 0x%08x", addr)
	}
	return nil
}

func (c *CPU) load(addr, size uint32) (uint32, error) {
	if err := c.check(addr, size); err != nil {
		return 0, err
	}
	switch size {
	case 1:
		return uint32(c.Mem[addr]), nil
	case 2:
		return uint32(binary.LittleEndian.Uint16(c.Mem[addr:])), nil
	}
	return binary.LittleEndian.Uint32(c.Mem[addr:]), nil
}

func (c *CPU) store(addr, size, v uint32) error {
	if err := c.check(addr, size); err != nil {
		return err
	}
	switch size {
	case 1:
		c.Mem[addr] = byte(v)
	case 2:
		binary.LittleEndian.PutUint16(c.Mem[addr:], uint16(v))
	default:
		binary.LittleEndian.PutUint32(c.Mem[addr:], v)
	}
	return nil
}
```

### Bagian 2, eksekusi instruksi
```
func (c *CPU) Step() error {
	raw, err := c.load(c.PC, 4)
	if err != nil {
		return err
	}
	i := Inst(raw)
	next := c.PC + 4

	switch i.Opcode() {
	case opLUI:
		c.setReg(i.Rd(), i.ImmU())

	case opAUIPC:
		c.setReg(i.Rd(), c.PC+i.ImmU())

	case opJAL:
		c.setReg(i.Rd(), next) // simpan alamat kembali
		next = c.PC + uint32(i.ImmJ())

	case opJALR:
		// hitung target dulu, karena rd bisa sama dengan rs1
		target := (c.Reg[i.Rs1()] + uint32(i.ImmI())) &^ 1
		c.setReg(i.Rd(), next)
		next = target

	case opBranch:
		take, err := c.branchTaken(i)
		if err != nil {
			return err
		}
		if take {
			next = c.PC + uint32(i.ImmB())
		}

	case opLoad:
		if err := c.execLoad(i); err != nil {
			return err
		}

	case opStore:
		if err := c.execStore(i); err != nil {
			return err
		}

	case opImm:
		sra := i.Funct7() == 0x20 // hanya berarti untuk SRAI
		res := alu(i.Funct3(), c.Reg[i.Rs1()], uint32(i.ImmI()), false, sra)
		c.setReg(i.Rd(), res)

	case opReg:
		alt := i.Funct7() == 0x20 // SUB dan SRA
		res := alu(i.Funct3(), c.Reg[i.Rs1()], c.Reg[i.Rs2()], alt, alt)
		c.setReg(i.Rd(), res)

	case opFence:
		// tidak melakukan apa pun pada emulator satu-core

	case opSystem:
		switch i.ImmI() {
		case 0: // ECALL
			if err := c.ecall(); err != nil {
				return err
			}
		case 1: // EBREAK
			c.Halted = true
		default:
			return fmt.Errorf("instruksi SYSTEM tidak didukung di PC=0x%08x", c.PC)
		}

	default:
		return fmt.Errorf("opcode tidak dikenal 0x%02x di PC=0x%08x", i.Opcode(), c.PC)
	}

	c.PC = next
	return nil
}
```

Fungsi-fungsi pendukung:
```
func b2u(b bool) uint32 {
	if b {
		return 1
	}
	return 0
}

// alu dipakai bersama oleh format R dan I-ALU.
func alu(f3, a, b uint32, sub, sra bool) uint32 {
	switch f3 {
	case 0: // ADD / SUB / ADDI
		if sub {
			return a - b
		}
		return a + b
	case 1: // SLL
		return a << (b & 31)
	case 2: // SLT
		return b2u(int32(a) < int32(b))
	case 3: // SLTU
		return b2u(a < b)
	case 4: // XOR
		return a ^ b
	case 5: // SRL / SRA
		if sra {
			return uint32(int32(a) >> (b & 31))
		}
		return a >> (b & 31)
	case 6: // OR
		return a | b
	}
	return a & b // AND
}

func (c *CPU) branchTaken(i Inst) (bool, error) {
	a, b := c.Reg[i.Rs1()], c.Reg[i.Rs2()]
	switch i.Funct3() {
	case 0: // BEQ
		return a == b, nil
	case 1: // BNE
		return a != b, nil
	case 4: // BLT
		return int32(a) < int32(b), nil
	case 5: // BGE
		return int32(a) >= int32(b), nil
	case 6: // BLTU
		return a < b, nil
	case 7: // BGEU
		return a >= b, nil
	}
	return false, fmt.Errorf("funct3 branch tidak valid")
}

func (c *CPU) execLoad(i Inst) error {
	addr := c.Reg[i.Rs1()] + uint32(i.ImmI())
	var v uint32
	var err error
	switch i.Funct3() {
	case 0: // LB (sign-extend)
		v, err = c.load(addr, 1)
		v = uint32(int32(int8(v)))
	case 1: // LH (sign-extend)
		v, err = c.load(addr, 2)
		v = uint32(int32(int16(v)))
	case 2: // LW
		v, err = c.load(addr, 4)
	case 4: // LBU
		v, err = c.load(addr, 1)
	case 5: // LHU
		v, err = c.load(addr, 2)
	default:
		return fmt.Errorf("funct3 load tidak valid")
	}
	if err != nil {
		return err
	}
	c.setReg(i.Rd(), v)
	return nil
}

func (c *CPU) execStore(i Inst) error {
	addr := c.Reg[i.Rs1()] + uint32(i.ImmS())
	v := c.Reg[i.Rs2()]
	switch i.Funct3() {
	case 0: // SB
		return c.store(addr, 1, v)
	case 1: // SH
		return c.store(addr, 2, v)
	case 2: // SW
		return c.store(addr, 4, v)
	}
	return fmt.Errorf("funct3 store tidak valid")
}
```

### Bagian 3, syscall
Program butuh cara berhenti dan mencetak teks. Kita pinjam konvensi Linux RISC-V: nomor syscall di a7 (x17), argumen di a0–a2 (x10–x12).
```
func (c *CPU) ecall() error {
	switch c.Reg[17] { // a7
	case 93:
		c.Halted = true
		c.ExitCode = int(int32(c.Reg[10]))
	case 64:
		buf, n := c.Reg[11], c.Reg[12]
		if err := c.check(buf, n); err != nil {
			return err
		}
		if c.Reg[10] == 1 {
			os.Stdout.Write(c.Mem[buf : buf+n])
		}
		c.Reg[10] = n // nilai kembalian: jumlah byte tertulis
	default:
		return fmt.Errorf("syscall tidak dikenal: %d", c.Reg[17])
	}
	return nil
}
```

## Program Utama (main.go)
```
package main

import (
	"fmt"
	"os"
)

const memSize = 1 << 20 // 1 MiB

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "pemakaian: rv32i program.bin")
		os.Exit(2)
	}
	prog, err := os.ReadFile(os.Args[1])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2)
	}

	cpu := NewCPU(memSize)
	copy(cpu.Mem, prog) // program dimuat di alamat 0, PC mulai 0

	for !cpu.Halted {
		if err := cpu.Step(); err != nil {
			fmt.Fprintf(os.Stderr, "error: %v (PC=0x%08x)\n", err, cpu.PC)
			os.Exit(1)
		}
	}
	os.Exit(cpu.ExitCode)
}
```

## Coba tes (test.go)
Kita coba rakit program kecil yang menghitung 5+7.

```
package main

import (
	"encoding/binary"
	"testing"
)

func TestAdd(t *testing.T) {
	prog := []uint32{
		0x00500093, // addi x1, x0, 5
		0x00700113, // addi x2, x0, 7
		0x002081B3, // add  x3, x1, x2
		0x05d00893, // addi a7, x0, 93   (syscall exit)
		0x00018513, // addi a0, x3, 0    (a0 = x3)
		0x00000073, // ecall
	}
	cpu := NewCPU(1024)
	for n, w := range prog {
		binary.LittleEndian.PutUint32(cpu.Mem[n*4:], w)
	}
	for !cpu.Halted {
		if err := cpu.Step(); err != nil {
			t.Fatal(err)
		}
	}
	if cpu.ExitCode != 12 {
		t.Fatalf("exit code = %d, harapan 12", cpu.ExitCode)
	}
}
```