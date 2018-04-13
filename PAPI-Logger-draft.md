General Logger interfaces:

```c++
// abstract interface that is called by Loggable objects
class Logger {
public:
    enum event_type {
        iteration_complete,
        // other event types,
        size
    };

    virtual void log(event_type event) const = 0;

    virtual ~Logger() = default;
};


// Inteface for recognizing loggable objects
class Loggable {
public:
    virtual void add_logger(std::shared_Ptr<const Logger> logger) = 0;

    virtual ~Loggable() = default;
};


// MixIn to quickly enable Logging for classes
template <typename ConcreteLogger>
class BasicLogger : public Loggable {

    void add_logger(std::shared_ptr<const Logger> logger) override {
        loggers.push_back(std::move(logger));
    }

    void log(event_type event) const {
        for(auto &logger : loggers) {
            logger->log(event);
        }
    }

protected:
    std::vector<std::shared_ptr<const Logger>> loggers;
};
```

Implementation of the `PapiLogger`:

```c++
// unfortunately, has to be singleton
class PapiLogger : public Logger {
public:
    void log(event_type event) override {
        if (event == event_type::iteration_complete) {
            ++num_iterations;
        }
        // handling for other events
    }

    static std::shared_ptr<PapiLogger> get_instance() {
        static std::shared_ptr<PapiLogger> instance{new PapiLogger()};
        return instance;
    }

private:
    PapiLogger() : handle_{papi_sde_init("ginkgo", event_type::size)} {
        if (handle_ == nullptr) {
            throw std::runtime_error("Unable to initialize PAPI");
        }
        const auto mode = /*something, report doesn't specify what*/;
        papi_sde_register_counter(handle_, "Number of Iterations",
                                  PAPI_SDE_LLINT, mode, &num_iterations);
        // register other counters
    }

    ~PapiLogger() {
        // is there a papi_sde_destroy()? Seems there should be, but is not listed.
    }

    void *handle_;
    mutable gko::int64 num_iterations;
    // counters for other events
};
```

A loggable object (e.g. a solver):

```c++
class Cg : public BasicLinOp<Cg>, public BasicLogger<Cg> {
public:
    void apply(...) {
    // setup
    for (int i = 0; i < num_iters; ++i) {
        // do iteration
        this->log(Logger::event_type::iteration_complete);
    }
};
```

And we add a logger like this:

```c++
auto factory = Cg::Factory::create(...);
auto cg = factory->generate(matrix);
cg->add_logger(PapiLogger::get_instance());
cg->add_logger(/* some other loger the user wants in addition to PAPI */);
cg->apply(b, x);  // will increment PapiLogger::num_iterations after every iteration
```