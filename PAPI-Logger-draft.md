TL; DR (or DU)
==============

Let __N__  be the number of _types_ of counters we want to support, and __C__ a small constant.
Then, adding PAPI support to Ginkgo (after Ginkgo's general logging facilities are set up), will cost:

*  If using `papi_sde_register_counter`: __C + 4N__ statements.
*  If using `papi_sde_log`: __C + 2N__ statements.

In both cases, the user has to add __1__ additional statement to enable PAPI support for a Ginkgo object in their application.

Implementation of `PapiLogger`
==============================

This is a simplified version that shows the differences between an implementation based on `papi_sde_register_counter` and `papi_sde_log`.

The code uses some of the internal Ginkgo's base classes, which do not depend on PAPI in any way.
For reference, the implementation of these classes is at the end of the page.


Version 1: using `papi_sde_register_event`
------------------------------------------

```c++
class PapiLogger : public Logger {
public:
    static std::unique_ptr<PapiLogger> create(void *handle)
    {
        return xstd::make_unique<PapiLogger>(handle);
    }

    void on_iteration_complete(/* parameters */) const override
    {
        ++num_iterations;
    }

    // implementation for on_<event_type> for other events
    // we export to PAPI - we don't need implementations for
    // events we do not export

protected:
    PapiLogger(void *handle) : handle_(handle)
    {
        const auto mode = /*something, report doesn't specify what*/;
        papi_sde_register_counter(handle_, "gko::num_iterations",
                                  PAPI_SDE_LLINT, mode, &num_iterations);
        // register other counters we export to PAPI
    }

private:
    void *handle_;
    mutable gko::int64 num_iterations;
    // counters for other events we export to PAPI
};
```

Version 2: using `papi_sde_log`
-------------------------------

```c++
class PapiLogger : public Logger {
public:
    static std::unique_ptr<PapiLogger> create(void *handle)
    {
        return xstd::make_unique<PapiLogger>(handle);
    }

    void on_iteration_complete(/* parameters */) const override
    {
        papi_sde_log(handle_, "gko::num_iterations");
    }

    // implementation for on_<event_type> for other events
    // we export to PAPI - we don't need implementations for
    // events we do not export

protected:
    PapiLogger(void *handle) : handle_(handle)
    {}

private:
    void *handle_;
};
```

Enable PAPI for a Ginkgo solver (user's code)
---------------------------------------------

```c++
auto mtx = /* get the system matrix */;
auto b = /* get RHS */;
auto x = /* get initial solution */;
void *handle = /* obtain a PAPI handle */;             # PAPI init
auto factory = Cg::Factory::create(/* parameters */);
auto cg = factory->generate(gko::give(matrix));
cg->add_logger(PapiLogger::create(handle));            # Add PAPI to solver
cg->apply(gko::lend(b), gko::lend(x));
```

The code is the same with both versions of the `PapiLogger`,
and there are no changes needed within the CG solver itself to support PAPI.



General `Logger`-related interfaces used in the above example
=============================================================

__Nothing related to PAPI in this section.__

```c++
// abstract interface that is called by Loggable objects
class Logger {
public:
    virtual ~Logger() = default;
    
    virtual void on_iteration_complete(/* parameters */) const {}
    
    // other on_<event_type> methods
};


// Inteface for recognizing loggable objects
class Loggable {
public:
    virtual ~Loggable() = default;
    
    virtual void add_logger(std::shared_Ptr<const Logger> logger) = 0;
};


// MixIn to easily enable Logging for different classes
template <typename ConcreteLogger>
class EnableLogging : public Loggable {
public:

    void add_logger(std::shared_ptr<const Logger> logger) override {
        loggers.push_back(std::move(logger));
    }

protected:
    void log_iteration_complete(/* parameters */) const {
        for(auto &logger : loggers) {
            logger->on_iteration_complete(/* parameters */);
        }
    }

    // other log_<event_type> methods

    std::vector<std::shared_ptr<const Logger>> loggers;
};
```

An example implementation of a loggable object (e.g. a solver):

```c++
class Cg : public BasicLinOp<Cg>, public EnableLogging<Cg> {
public:
    void apply(/* parameters */) {
        // setup
        for (int i = 0; i < num_iters; ++i) {
            // do iteration
            this->log_iteration_complete(/* parameters */); // doesn't have to know about PAPI
        }
    }
};
```

