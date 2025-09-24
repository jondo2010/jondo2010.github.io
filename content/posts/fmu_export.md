+++
title = "FMU Export in Rust"
date = 2025-09-24
+++

# Generate FMUs with rust-fmi

Back when I updated the [rust-fmi](https://github.com/jondo2010/rust-fmi) library in early 2024 to support FMI 3.0, I did it mostly out of interest in learning the new standard and its features. I hadn't had access to a commercial modeling tool such as Dymola for a while, and since OpenModelica didn't (and still doesn't) support FMI3 export, I was limited at the time to testing the new import code with the Modelica [reference FMUs](https://github.com/modelica/Reference-FMUs).

Even at the beginning of the project several years earlier, I thought it would be cool to also support the creation of FMUs. I didn't have a clear idea about what that would look like, or what a direct Rust implementation would offer over existing tools, and actually my experience with Rust wasn't as extensive then. Now with the recent addition of the FMI layered standards such as [fmi-ls-bus](https://modelica.github.io/fmi-ls-bus/main), I started imagining how Rust-based FMUs could be used in a heterogeneous HIL/SIL setup, with real controllers and other components implemented in Rust, as well as e.g., gateway FMUs connecting to real CAN or Ethernet networks.

After a few weeks of evening and weekend work, I'm excited to announce _experimental_ support for **generating and exporting FMUs** (Functional Mockup Units) directly from Rust code using [rust-fmi](https://github.com/jondo2010/rust-fmi). With both **export** and **import** capabilities, this brings the power of Rust to the FMI world, offering a memory-safe alternative to modeling tools and C/C++ implementations.

{% dialog(title="What is an FMU?") %}
An FMU (Functional Mockup Unit) is a self-contained, cross-platform, tool-agnostic simulation component (shared library plus XML metadata) defined by the FMI standard, enabling standardized integration of models across tools for system integration, hardware‑in‑the‑loop testing, protected model exchange, co-simulation, and digital twin scenarios; for Rust developers it provides a path to plug safe, performant Rust-based controllers and algorithms into established engineering workflows in automotive, aerospace, and industrial automation without exposing source code.
{% end %}

## Existing development

Generating FMUs is typically accomplished with one of two approaches:

1. **Model-based tools** - While powerful, the best here require expensive commercial licenses
2. **Manual C/C++ implementation** - This requires writing substantial boilerplate code, manually authoring XML model descriptions, and managing complex build processes

## 🦀 Rust on the scene

With `rust-fmi`'s new export capabilities, you can now define your entire FMU model in a single Rust struct using derive macros. The library automatically generates:

- ✅ **Full FMI C API implementation** - All the low-level bindings handled for you  
- ✅ **Type-safe variable handling** - Rust's type system prevents common errors
- ✅ **Complete XML model description** - No manual XML authoring required
- ✅ **Memory-safe implementation** - No more worrying about C-style memory management
- ✅ **FMU packaging tooling** - Easily bundle your FMU with `cargo xtask bundle`

## A Simple Example

Here's how easy it is to re-create the VanDerPol oscillator FMU in Rust, complete with array state variables:

```rust
/// Van der Pol oscillator FMU model
///
/// This is implemented as a system of first-order ODEs:
/// - der(x0) = x1  
/// - der(x1) = μ(1 - x0²)x1 - x0
#[derive(FmuModel, Default, Debug)]
struct VanDerPol {
    /// State variables (x[0], x[1])
    #[variable(causality = Output, variability = Continuous, state, start = [2.0, 0.0], initial = Exact)]
    x: [f64; 2],

    /// Derivatives of state variables (der(x[0]), der(x[1]))
    #[variable(causality = Local, variability = Continuous, derivative = x, initial = Calculated)]
    der_x: [f64; 2],

    /// Scalar parameter μ
    #[variable(causality = Parameter, variability = Fixed, start = 1.0, initial = Exact)]
    mu: f64,
}

impl UserModel for VanDerPol {
    type LoggingCategory = DefaultLoggingCategory;

    fn calculate_values(&mut self, _context: &ModelContext<Self>) -> Result<Fmi3Res, Fmi3Error> {
        // Calculate the derivatives according to Van der Pol equations:
        // der(x[0]) = x[1]
        self.der_x[0] = self.x[1];

        // der(x[1]) = mu * ((1 - x[0]²) * x[1]) - x[0]
        self.der_x[1] = self.mu * ((1.0 - self.x[0] * self.x[0]) * self.x[1]) - self.x[0];

        Ok(Fmi3Res::OK)
    }
}

// Export the complete FMU with C API
fmi_export::export_fmu!(VanDerPol);
```

That's it! This single Rust file defines a complete, functional FMU that can be used in any FMI-compliant simulation environment.

This code generator creates the following `modelDescription.xml` for you automatically:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmiModelDescription fmiVersion="3.0" modelName="vanderpol" instantiationToken="ec737b5d-a92d-5527-9670-10230e0879f7" description="Van der Pol oscillator FMU example ported from Reference FMUs" version="0.1.0" generationTool="rust-fmi" generationDateAndTime="2025-09-10T14:46:05.166472+00:00">
  <ModelExchange modelIdentifier="vanderpol" canGetAndSetFMUState="false" canSerializeFMUState="false"/>
  <DefaultExperiment startTime="0" stopTime="20" stepSize="0.01"/>
  <ModelVariables>
    <Float64 name="time" valueReference="0" causality="independent" variability="continuous" description="Simulation time"/>
    <Float64 start="2 0" initial="exact" name="x" valueReference="1" causality="output" variability="continuous">
      <Dimension start="2"/>
    </Float64>
    <Float64 initial="calculated" name="der_x" valueReference="2" variability="continuous">
      <Dimension start="2"/>
    </Float64>
    <Float64 start="1" initial="exact" name="mu" valueReference="3" causality="parameter" variability="fixed"/>
  </ModelVariables>
  <ModelStructure>
    <Output valueReference="1"/>
    <ContinuousStateDerivative valueReference="2"/>
    <InitialUnknown valueReference="2"/>
  </ModelStructure>
</fmiModelDescription>
```

## Key Benefits

- **Single Source of Truth:** Define your model, variables, and metadata in one Rust struct using attributes.
- **Type Safety:** Rust checks array bounds and derivative links at compile time, reducing runtime errors.
- **Efficient Code:** The derive macro generates code with no extra runtime cost.
- **Simplified Workflow:** No manual XML or syncing between code and metadata—array variables are handled for you.
- **Modern Tooling:** Use Cargo, Rust’s testing tools, and IDE support out of the box.

## Architecture Highlights
The implementation leverages several key components:

- `#[derive(FmuModel)]` macro: Automatically analyzes your struct and generates all required FMI infrastructure
- Variable attributes: Declarative syntax for specifying FMI variable properties
- Array support: Native handling of Rust arrays with automatic FMI variable generation
- Derivative linking: Automatic setup of state-derivative relationships
- `UserModel` trait: Clean separation between generated code and your custom model logic
- Support for all FMI 3.0 variable types and causality/variability options
- Event handling and state management hooks (demonstrated in `BouncingBall` and `Stair` examples)

## Current Limitations
This feature is experimental with some important caveats:

- FMI 3.0 only: FMI 2.0 support could also be added in the future
- **Model-Exchange** only: **Co-Simulation** and **Scheduled-Execution** support coming later
- Only fixed-size arrays (also multi-dimensional) are supported currently. Support for `Vec<T>` and tensor types such as `ndarray::Array` could definitely be supported in the future.
- Limited validation: While basic checks are in place, more comprehensive validation of model structure and variable properties is needed.
- Lots of not-yet-implemented FMI features: e.g., directional derivatives, configuration mode, custom logging categories, etc.

## Kicking the Tires

Check out the examples of ports of reference FMUs like the `VanDerPol`, `Dahlquist`, `BouncingBall` and `Stair`. You can build them with Cargo:
```bash
cargo xtask bundle --package bouncing_ball
```

And test using either your favorite FMI importer or the included `fmi-sim` tool:
```bash
cargo run --package fmi-sim -- --model target/fmu/bouncing_ball.fmu model-exchange
```

## The Future

Depending on feedback, interest, and my own time, I'd like to add support for:

- fmi-ls-bus and other layered standards
- Co-Simulation and Scheduled-Execution interfaces  
- Dynamically-sized arrays and tensors
- Advanced features like directional derivatives
- FMI 2.0 compatibility

## Community

The `rust-fmi` project is open source and actively seeking contributors. Whether you're a simulation expert, Rust developer, or FMI user, I'd love your feedback and contributions.

- **GitHub**: [jondo2010/rust-fmi](https://github.com/jondo2010/rust-fmi)
- **Documentation**: [docs.rs/fmi-export](https://docs.rs/fmi-export)
- **Issues**: Report bugs and request features on GitHub

---
*The rust-fmi project is bringing Rust to the Modelica and FMI community. Try it out and let me know what you think!*