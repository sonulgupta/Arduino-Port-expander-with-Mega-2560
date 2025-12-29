Step 1: Update the Header (arduino_port_expander.h)You need to remove the single instance pointer and allow each expander to be identified. The simplest way is to keep a pointer for each one.Modify the class variables:C++class ArduinoPortExpander : public Component, public I2CDevice {
public:
  // Remove: static ArduinoPortExpander *instance;
  // Add these instead:
  static ArduinoPortExpander *exp1;
  static ArduinoPortExpander *exp2;

  ArduinoPortExpander(I2CBus *bus, uint8_t address, int unit_id = 1) {
    set_i2c_address(address);
    set_i2c_bus(bus);
    
    // Assign to the correct static pointer based on an ID
    if (unit_id == 1) exp1 = this;
    if (unit_id == 2) exp2 = this;
  }
  // ... rest of code
};

// Initialize them at the bottom of the file
ArduinoPortExpander *ArduinoPortExpander::exp1 = nullptr;
ArduinoPortExpander *ArduinoPortExpander::exp2 = nullptr;
Step 2: Update the YAML InitializationWhen you create the custom components, pass the "Unit ID" (1 or 2) to the constructor so the header knows which pointer to assign.YAMLcustom_component:
  - lambda: |-
      auto ape1 = new ArduinoPortExpander(id(i2c_bus_1), 0x08, 1); // Unit 1
      return {ape1};
    id: expander1

  - lambda: |-
      auto ape2 = new ArduinoPortExpander(id(i2c_bus_1), 0x09, 2); // Unit 2 (different address)
      return {ape2};
    id: expander2
Step 3: Use them in LambdasIn your sensors or outputs, you now simply reference the specific pointer (exp1 or exp2) you need.For Expander 1:YAMLbinary_sensor:
  - platform: custom
    lambda: |-
      return []() -> std::vector<binary_sensor::BinarySensor*> {
        if (ArduinoPortExpander::exp1 == nullptr) return {}; 
        return {
          ArduinoPortExpander::exp1->get_binary_sensor(0),
          ArduinoPortExpander::exp1->get_binary_sensor(1)
        };
      }();
For Expander 2:YAMLbinary_output:
  - platform: custom
    type: binary
    lambda: |-
      return []() -> std::vector<esphome::output::BinaryOutput*> {
        if (ArduinoPortExpander::exp2 == nullptr) return {}; 
        return {
          ArduinoPortExpander::exp2->get_binary_output(48),
          ArduinoPortExpander::exp2->get_binary_output(49)
        };
      }();
Important ConsiderationsI2C Addresses: Ensure each physical Arduino/Expander has a unique I2C address (e.g., 0x08 and 0x09). You must set this in the Arduino code for the expanders as well.Bus Contention: With two expanders and many pins, the I2C bus will be busy. If you notice "timeout" errors in your logs, ensure you have strong pull-up resistors (around $4.7\text{k}\Omega$) on the SDA and SCL lines.
