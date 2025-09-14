// ==================================================
// category_theory_ops.cpp - Symbolic Category Theory Operations
// ==================================================
// This plugin allows for the symbolic representation of objects, morphisms,
// and composition within the woflang environment.
//
// Assumes the WofValue type can hold and differentiate std::string values,
// likely for symbols or custom string types, as suggested by symbolic_engine.hpp.

#include "core/woflang.hpp"
#include <iostream>
#include <string>
#include <vector>
#include <stdexcept>
#include <sstream>

// Helper function to safely get a string from a WofValue.
// NOTE: This assumes a WofValue structure that can hold strings.
// You will need to adapt this to your actual woflang::WofValue implementation.
// For this example, we'll assume it has a method like `as_string()`.
// If it only uses `.d`, this plugin would need a core engine update.
static std::string get_string_from_value(const woflang::WofValue& val, const char* context) {
    // This is a placeholder for your actual implementation.
    // For demonstration, we'll convert the double to a string.
    // In a real symbolic engine, you'd extract the actual string value.
    if (val.s) { // Assuming a string member `s`
        return *val.s;
    }
    throw std::runtime_error(std::string(context) + ": expected a string/symbol value.");
}

// Represents a parsed morphism: f: A -> B
struct Morphism {
    std::string name;
    std::string source;
    std::string target;

    // Deserializes a string like "f: A -> B" into a Morphism struct.
    static Morphism from_string(const std::string& s) {
        Morphism m;
        size_t colon_pos = s.find(':');
        size_t arrow_pos = s.find("->");

        if (colon_pos == std::string::npos || arrow_pos == std::string::npos) {
            throw std::runtime_error("Invalid morphism format. Expected 'name: source -> target'");
        }

        m.name = s.substr(0, colon_pos);
        m.source = s.substr(colon_pos + 2, arrow_pos - (colon_pos + 3));
        m.target = s.substr(arrow_pos + 3);
        return m;
    }

    // Serializes the struct back into a string.
    std::string to_string() const {
        return name + ": " + source + " -> " + target;
    }
};


extern "C" {

#ifndef WOFLANG_PLUGIN_EXPORT
#  ifdef _WIN32
#    define WOFLANG_PLUGIN_EXPORT extern "C" __declspec(dllexport)
#  else
#    define WOFLANG_PLUGIN_EXPORT extern "C"
#  endif
#endif

WOFLANG_PLUGIN_EXPORT void init_plugin(woflang::WoflangInterpreter::OpTable* op_table) {

    // op: identity
    // ( obj -- id_obj )
    // Creates an identity morphism for a given object.
    // Example: "A" identity -> "id_A: A -> A"
    (*op_table)["identity"] = [](std::stack<woflang::WofValue>& stack) {
        if (stack.empty()) {
            throw std::runtime_error("identity requires one object from the stack.");
        }
        std::string obj_name = get_string_from_value(stack.top(), "identity");
        stack.pop();

        Morphism id_morphism = {"id_" + obj_name, obj_name, obj_name};
        
        woflang::WofValue result;
        result.s = std::make_shared<std::string>(id_morphism.to_string());
        stack.push(result);
    };

    // op: compose
    // ( morphism1 morphism2 -- new_morphism )
    // Composes two morphisms, g and f, into (g . f).
    // Example: "f: A -> B" "g: B -> C" compose -> "(g . f): A -> C"
    (*op_table)["compose"] = [](std::stack<woflang::WofValue>& stack) {
        if (stack.size() < 2) {
            throw std::runtime_error("compose requires two morphisms from the stack.");
        }
        std::string g_str = get_string_from_value(stack.top(), "compose"); stack.pop();
        std::string f_str = get_string_from_value(stack.top(), "compose"); stack.pop();

        Morphism g = Morphism::from_string(g_str);
        Morphism f = Morphism::from_string(f_str);

        if (f.target != g.source) {
            throw std::runtime_error("Composition failed: target of first morphism ('" + f.target + "') does not match source of second ('" + g.source + "').");
        }

        Morphism composed = {"(" + g.name + " . " + f.name + ")", f.source, g.target};

        woflang::WofValue result;
        result.s = std::make_shared<std::string>(composed.to_string());
        stack.push(result);
    };

    // op: source
    // ( morphism -- obj )
    // Pushes the source object of a morphism onto the stack.
    // Example: "f: A -> B" source -> "A"
    (*op_table)["source"] = [](std::stack<woflang::WofValue>& stack) {
        if (stack.empty()) {
            throw std::runtime_error("source requires a morphism from the stack.");
        }
        std::string m_str = get_string_from_value(stack.top(), "source");
        stack.pop();

        Morphism m = Morphism::from_string(m_str);

        woflang::WofValue result;
        result.s = std::make_shared<std::string>(m.source);
        stack.push(result);
    };
    
    // op: target
    // ( morphism -- obj )
    // Pushes the target object of a morphism onto the stack.
    // Example: "f: A -> B" target -> "B"
    (*op_table)["target"] = [](std::stack<woflang::WofValue>& stack) {
        if (stack.empty()) {
            throw std::runtime_error("target requires a morphism from the stack.");
        }
        std::string m_str = get_string_from_value(stack.top(), "target");
        stack.pop();

        Morphism m = Morphism::from_string(m_str);

        woflang::WofValue result;
        result.s = std::make_shared<std::string>(m.target);
        stack.push(result);
    };
}

} // extern "C"
