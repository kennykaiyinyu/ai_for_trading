# ai_for_trading

#include <variant>
#include <vector>
#include <string>
#include <iostream>
#include <cmath>
#include <algorithm>

// Error type with enum for efficiency
enum class ErrorCode { EmptyInput, UnsortedX, InvalidValue, OutOfBounds, InvalidInput };
struct Error {
    ErrorCode code;
    static std::string message(ErrorCode c) {
        switch (c) {
            case ErrorCode::EmptyInput: return "Empty input";
            case ErrorCode::UnsortedX: return "X coordinates not sorted";
            case ErrorCode::InvalidValue: return "Invalid coordinate: NaN or infinity";
            case ErrorCode::OutOfBounds: return "X' out of bounds";
            case ErrorCode::InvalidInput: return "Invalid X' input";
            default: return "Unknown error";
        }
    }
};

// Visitor for variant handling
struct ResultVisitor {
    void operator()(const Error& e) const {
        std::cerr << "Error: " << Error::message(e.code) << '\n';
        // Log to file/monitoring system
    }
    template<typename T>
    void operator()(const T& value) const {
        std::cout << "Success: " << value << '\n';
    }
};

class Curve {
    const std::vector<double> x_coords, y_coords; // Immutable, non-empty, sorted x
private:
    Curve(const std::vector<double>& x, const std::vector<double>& y)
        : x_coords(x), y_coords(y) {}

public:
    using Result = std::variant<Curve, Error>;
    template<typename T> using MethodResult = std::variant<T, Error>;

    // Factory: validates input (non-empty, equal sizes, sorted x, finite values)
    static Result create(const std::vector<double>& x, const std::vector<double>& y) {
        if (x.empty() || x.size() != y.size()) return Error{ErrorCode::EmptyInput};
        if (!std::is_sorted(x.begin(), x.end())) return Error{ErrorCode::UnsortedX};
        for (size_t i = 0; i < x.size(); ++i) {
            if (!std::isfinite(x[i]) || !std::isfinite(y[i])) {
                return Error{ErrorCode::InvalidValue};
            }
        }
        return Curve(x, y);
    }

    // Lookup: interpolates y' for x', handles invalid x'
    MethodResult<double> lookup(double x_prime) const {
        if (!std::isfinite(x_prime)) return Error{ErrorCode::InvalidInput};
        if (x_prime < x_coords.front() || x_prime > x_coords.back()) {
            return Error{ErrorCode::OutOfBounds};
        }

        // Find interval [x_i, x_{i+1}] where x_i <= x_prime <= x_{i+1}
        auto it = std::lower_bound(x_coords.begin(), x_coords.end(), x_prime);
        if (it == x_coords.begin()) return y_coords[0]; // Exact match at start
        size_t i = std::distance(x_coords.begin(), it) - 1;

        // Linear interpolation
        double x0 = x_coords[i], x1 = x_coords[i + 1];
        double y0 = y_coords[i], y1 = y_coords[i + 1];
        double t = (x_prime - x0) / (x1 - x0);
        return y0 + t * (y1 - y0);
    }

    // Safe methods: direct returns
    size_t get_size() const { return x_coords.size(); }
    const std::vector<double>& get_x() const { return x_coords; }
    const std::vector<double>& get_y() const { return y_coords; }
};

int main() {
    // Create curve
    auto result = Curve::create({1.0, 2.0, 3.0}, {10.0, 20.0, 30.0});
    if (!std::holds_alternative<Curve>(result)) {
        std::visit(ResultVisitor{}, result);
        return 1;
    }
    Curve curve = std::get<Curve>(std::move(result));

    // Test lookup
    auto tests = {
        curve.lookup(2.5),        // Valid: interpolate
        curve.lookup(0.0),        // Out-of-range
        curve.lookup(std::nan("")) // Invalid
    };
    for (const auto& r : tests) {
        std::visit(ResultVisitor{}, r);
    }

    // Safe methods
    std::cout << "Size: " << curve.get_size() << '\n'; // Prints "Size: 3"

    return 0;
}
