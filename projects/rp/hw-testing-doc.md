---
layout: default
title: Dokumentácia k HW Testing Library
---

# Hardware Testing Library

Hardware Testing Library is a Python library which utilizes debugging over `gdb` connected to the board being tested over SWD to directly manipulate registers of memory-mapped peripherals of the MCU on the board. Currently only STM32 MCUs are supported. This can be used to easily write tests that can verify whether the newly-manufactured hardware is working properly.

## Motivation

Testing is important. However, we were lacking an automated way of testing newly produced boards for various components of the project. Firmware is unable to test all functions of the board, because we cannot isolate hardware from the software problems in that case. We needed something that can selectively test every testable function of a board and report failures before the firmware is loaded to the board.

When we are connected to the MCU on the board under test through SWD and `gdb`, we are able to directly control peripherals of the MCU and therefore test the board. However, doing this manually is a very tedious work as one has to keep track of every address of every peripheral register. So this tool was created. It can keep track of addresses of peripheral registers and provide easy access to them. It can also launch various tests in a sequence and report results. This allowed us to automate the process of testing hardware boards.

## How it works

The board under test is connected to the PC using SWD (Serial Wire Debug) debugging interface. `stlink` utility then provides a `gdb` server using which the MCU can be directly controlled. `gdb` is scriptable in Python (when the Python interpreter is invoked from the `gdb`). So the `arm-none-eabi-gdb` is started. It connects to the `gdb` server provided by the `stlink`, thereby taking control of the MCU. From within the `gdb` this tests written using this library are invoked. Those scripts are then controlling the MCU, activating peripherals and using them to test hardware connected to the MCU.

### Device object

Device object is a way this library keeps track of addresses of various peripheral registers. Each used MCU must first be defined. This is done in folder `devices/` of the source tree. Each MCU has its own file there.

In this file peripherals are defined. For each peripheral there is a class (subclass of `MMPeripheral` (Memory-Mapped Peripheral)). This class has a list called `fields`. This is a list of tuples `(type, register_name)`. `type` defines size of the register and may be one of the `T.uint8_t`, `T.uint16_t`, `T.uint32_t` (or `T.intXX_t` versions of those). `register_name` is the name of the register as specified in the datasheet or `None` for the reserved memory space.
Order of registers in this list is important, as each register has a specified offset from the given address of the peripheral. This offset is computed from sizes of registers that are in the list before the given register.

Where appropriate, there are also dictionaries `REG_NAME_bits` which define names of bits in register `REG_NAME`.

Example: Reset and Clock Control peripheral definition

{% highlight python %}
class RCC(MMPeripheral):
    fields = [(T.uint32_t,  'CR'),
              (T.uint32_t,  'CFGR'),
              (T.uint32_t,  'CIR'),
              (T.uint32_t,  'APB2RSTR'),
              (T.uint32_t,  'APB1RSTR'),
              (T.uint32_t,  'AHBENR'),
              (T.uint32_t,  'APB2ENR'),
              (T.uint32_t,  'APB1ENR'),
              (T.uint32_t,  'BDCR'),
              (T.uint32_t,  'CSR'),
              (T.uint32_t,  'AHBRSTR'),
              (T.uint32_t,  'CFGR2'),
              (T.uint32_t,  'CFGR3'),
              (T.uint32_t,  'CR2')]

    AHBENR_bits = {'TSCEN': 24,
                   'IOPFEN': 22,
                   'IOPEEN': 21,
                   'IOPDEN': 20,
                   'IOPCEN': 19,
                   'IOPBEN': 18,
                   'IOPAEN': 17,
                   'CRCEN': 6,
                   'FLITFEN': 4,
                   'SRAMEN': 2,
                   'DMAEN': 0}
{% endhighlight %}

Now that we have defined classes of peripherals we must create their instances. One type of peripheral may have more instances (for example there are several GPIOs on the STM32 MCU).

To do this, a class representing the MCU itself must be created. During initialization of this class peripherals are declared. As parameter an address is given, this is the address to which the peripheral is mapped. All those data can be found in the datasheet.

Example:

{% highlight python %}
class STM32F0(object):

    PERIPHERALS_BASE = 0x40000000

    APB_BUS_BASE = PERIPHERALS_BASE
    AHB1_BUS_BASE = PERIPHERALS_BASE + 0x00020000
    AHB2_BUS_BASE = PERIPHERALS_BASE + 0x08000000

    def __init__(self, device_memory):
        self.GPIO = {'A': GPIO(self.AHB2_BUS_BASE + 0x00000000, device_memory),
                     'B': GPIO(self.AHB2_BUS_BASE + 0x00000400, device_memory),
                     'C': GPIO(self.AHB2_BUS_BASE + 0x00000800, device_memory),
                     'D': GPIO(self.AHB2_BUS_BASE + 0x00000C00, device_memory),
                     'E': GPIO(self.AHB2_BUS_BASE + 0x00001000, device_memory),
                     'F': GPIO(self.AHB2_BUS_BASE + 0x00001400, device_memory)}
        self.RCC = RCC(self.AHB1_BUS_BASE + 0x00001000, device_memory)
{% endhighlight %}

### Mempoke (memory-poker): the magic

In the heart of this library is the 'memory-poker' (`mempoke`) module. It has two jobs: provide an r/w interface to the memory of the MCU and provide the base class `MMPeripheral` for defining and using Memory-Mapped Peripherals.

#### `device_memory`

To instantiate a MCU class an instance of `device_memory` is needed. This object provides direct read/write access to the memory of the MCU. It is provided by the `DeviceMemory` class in the `mempoke`. This class, when instantiated, will establish a connection to the `gdb` and therefore will be capable of modifying memory of the MCU.

#### `MMPeripheral`

The mother of all peripherals. This class is special in that assignment and reading operators are redefined. When something that subclasses this class is instantiated its field `fields` is taken. Address of the peripheral is supplied in the constructor and together with the offsets defined in the `fields` field of the subclass exact addresses of registers are computed.

Every assignment to a field of this class with the same name as some register will cause this assignment to be made in the memory of the MCU itself. Every reading of field will cause the memory of the MCU to be read.

Example:

{% highlight python %}
dev = STM32F0(mempoke.DeviceMemory())
dev.RCC.AHBENR |= (1 << dev.RCC.AHBENR_bits["IOPAEN"])
{% endhighlight %}

The above statement will cause this assignment to be done in the memory of the MCU itself (in effect, in this particular case, turning on GPIO A).

This is the API tests use to control the MCU. Simple and powerful.

### Test sequencer

Now we know how to control the MCU, let's write some tests. Test is a function. It must have a docstring where the first line briefly describes the test, then an empty line, and then a detailed explanation of the test. One test function should test only one feature of the hardware. It must be capable of receiving MCU with peripherals in an undefined state and it must leave them "cleaned up" (resetted). This will ensure that the test does not interfere with other tests, and that badly-behaving test will not interfere with this test.

Example:

{% highlight python %}
def test_usart_receive():
    """Tests USART Rx.
    This test verifies whether Controller is able to send data to the Reader. It assesses functionality of
    the IC4 and J1, as well as integrity of USART Rx signal path.
    If this test succeeds and there is no contact problem on the Tx path it is reasonable to assume that USART Tx
    will be working as well, as it cannot be tested directly.
    """
    reset_peripherals() # Reset peripherals, they may be in an undefined state

    ... # Test body

    reset_peripherals() # We must leave the MCU in a clean state
{% endhighlight %}

(Note: function `reset_peripherals()` in the above example must be implemented by author of tests.)

List of this functions is then given as a parameter to the `test_sequencer`, which runs them, informs user about the status and collects results. It also has several helper functions (for informative output, asking user some questions, etc...) which are documented in the `test_sequencer` module.

Testing is started by calling function `run([tests])`.
