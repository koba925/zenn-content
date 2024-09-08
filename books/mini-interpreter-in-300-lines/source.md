---
title: "付録：全ソースコード"
---

```py
class Scanner:
    def __init__(self, source) -> None:
        self._source = source
        self._current_position = 0

    def next_token(self):
        while self._current_char().isspace(): self._current_position += 1

        start = self._current_position
        match self._current_char():
            case "$EOF": return "$EOF"
            case c if c.isalpha():
                while self._current_char().isalnum() or self._current_char() == "_":
                    self._current_position += 1
                token = self._source[start:self._current_position]
                match token:
                    case "true": return True
                    case "false": return False
                    case _: return token
            case c if c.isnumeric():
                while self._current_char().isnumeric():
                    self._current_position += 1
                return int(self._source[start:self._current_position])
            case _:
                self._current_position += 1
                return self._source[start:self._current_position]

    def _current_char(self):
        if self._current_position < len(self._source):
            return self._source[self._current_position]
        else:
            return "$EOF"

class Parser:
    def __init__(self, source):
        self.scanner = Scanner(source)
        self._current_token = ""
        self._next_token()

    def parse_program(self):
        program: list = ["program"]
        while self._current_token != "$EOF":
            program.append(self._parse_statement())
        return program

    def _parse_statement(self):
        match self._current_token:
            case "{": return self._parse_block()
            case "var" | "set": return self._parse_var_set()
            case "if": return self._parse_if()
            case "while": return self._parse_while()
            case "def": return self._parse_def()
            case "return": return self._parse_return()
            case "print": return self._parse_print()
            case _: return self._parse_expression_statement()

    def _parse_block(self):
        block: list = ["block"]
        self._next_token()
        while self._current_token != "}":
            block.append(self._parse_statement())
        self._next_token()
        return block

    def _parse_var_set(self):
        op = self._current_token
        self._next_token()
        name = self._parse_primary()
        assert isinstance(name, str),  f"Expected a name, found `{name}`."
        self._consume_token("=")
        value = self._parse_expression()
        self._consume_token(";")
        return [op, name, value]

    def _parse_if(self):
        self._next_token()
        cond = self._parse_expression()
        self._check_token("{")
        conseq = self._parse_block()
        alt = ["block"]
        if self._current_token == "elif":
            alt = self._parse_if()
        elif self._current_token == "else":
            self._next_token()
            self._check_token("{")
            alt = self._parse_block()
        return ["if", cond, conseq, alt]

    def _parse_while(self):
        self._next_token()
        cond = self._parse_expression()
        self._check_token("{")
        body = self._parse_block()
        return ["while", cond, body]

    def _parse_def(self):
        self._next_token()
        name = self._parse_primary()
        assert isinstance(name, str),  f"Expected a name, found `{name}`."
        params = self._parse_parameters()
        body = self._parse_block()
        return ["var", name, ["func", params, body]]

    def _parse_return(self):
        self._next_token()
        value = 0
        if self._current_token != ";": value = self._parse_expression()
        self._consume_token(";")
        return ["return", value]

    def _parse_print(self):
        self._next_token()
        expr = self._parse_expression()
        self._consume_token(";")
        return ["print", expr]

    def _parse_expression_statement(self):
        expr = self._parse_expression()
        self._consume_token(";")
        return ["expr", expr]

    def _parse_expression(self):
        return self._parse_equality()

    def _parse_equality(self): return self._parse_binop_left(("=", "#"), self._parse_add_sub)
    def _parse_add_sub(self): return self._parse_binop_left(("+", "-"), self._parse_mult_div)
    def _parse_mult_div(self): return self._parse_binop_left(("*", "/"), self._parse_power)

    def _parse_binop_left(self, ops, sub_element):
        result = sub_element()
        while (op := self._current_token) in ops:
            self._next_token()
            result = [op, result, sub_element()]
        return result

    def _parse_power(self):
        power = self._parse_call()
        if self._current_token != "^": return power
        self._next_token()
        return ["^", power, self._parse_power()]

    def _parse_call(self):
        call = self._parse_primary()
        while self._current_token == "(":
            self._next_token()
            args = []
            while self._current_token != ")":
                args.append(self._parse_expression())
                if self._current_token != ")":
                    self._consume_token(",")
            call = [call] + args
            self._consume_token(")")
        return call

    def _parse_primary(self):
        match self._current_token:
            case "(":
                self._next_token()
                exp = self._parse_expression()
                self._consume_token(")")
                return exp
            case "func": return self._parse_func()
            case int(value) | bool(value) | str(value):
                self._next_token()
                return value
            case unexpected: assert False, f"Unexpected token `{unexpected}`."

    def _parse_func(self):
        self._next_token()
        params = self._parse_parameters()
        body = self._parse_block()
        return ["func", params, body]

    def _parse_parameters(self):
        self._consume_token("(")
        params = []
        while self._current_token != ")":
            param = self._current_token
            assert isinstance(param, str), f"Name expected, found `{param}`."
            self._next_token()
            params.append(param)
            if self._current_token != ")":
                self._consume_token(",")
        self._consume_token(")")
        return params

    def _consume_token(self, expected_token):
        self._check_token(expected_token)
        return self._next_token()

    def _check_token(self, expected_token):
        assert self._current_token == expected_token, \
               f"Expected `{expected_token}`, found `{self._current_token}`."

    def _next_token(self):
        self._current_token = self.scanner.next_token()
        return self._current_token

class Return(Exception):
    def __init__(self, value): self.value = value

class Environment:
    def __init__(self, parent:"Environment | None"=None):
        self._values = {}
        self._parent = parent

    def define(self, name, value):
        assert name not in self._values, f"`{name}` already defined."
        self._values[name] = value

    def assign(self, name, value):
        if name in self._values: self._values[name] = value
        elif self._parent is not None: self._parent.assign(name, value)
        else: assert False, f"`{name}` not defined."

    def get(self, name):
        if name in self._values: return self._values[name]
        if self._parent is not None: return self._parent.get(name)
        assert False, f"`{name}` not defined."

    def list(self):
        if self._parent is None: return [self._values]
        return self._parent.list() + [self._values]

class Evaluator:
    def __init__(self):
        self.output = []
        self._env = Environment()
        self._env.define("less", lambda a, b: a < b)
        self._env.define("print_env", self._print_env)

    def _print_env(self):
        for values in self._env.list():
            print({ k: self._to_print(v) for k, v in values.items() })

    def eval_program(self, program):
        self.output = []
        match program:
            case ["program", *statements]:
                for statement in statements:
                    self._eval_statement(statement)
            case unexpected: assert False, f"Internal Error at `{unexpected}`."

    def _eval_statement(self, statement):
        match statement:
            case ["block", *statements]: self._eval_block(statements)
            case ["var", name, value]: self._eval_var(name, value)
            case ["set", name, value]: self._eval_set(name, value)
            case ["if", cond, conseq, alt]: self._eval_if(cond, conseq, alt)
            case ["while", cond, body]: self._eval_while(cond, body)
            case ["return", value]: raise Return(self._eval_expr(value))
            case ["print", expr]: self._eval_print(expr)
            case ["expr", expr]: self._eval_expr(expr)
            case unexpected: assert False, f"Internal Error at `{unexpected}`."

    def _eval_block(self, statements):
        parent_env = self._env
        self._env = Environment(parent_env)
        for statement in statements:
            self._eval_statement(statement)
        self._env = parent_env

    def _eval_var(self, name, value):
        self._env.define(name, self._eval_expr(value))

    def _eval_set(self, name, value):
        self._env.assign(name, self._eval_expr(value))

    def _eval_if(self, cond, conseq, alt):
        if self._eval_expr(cond):
            self._eval_statement(conseq)
        else:
            self._eval_statement(alt)

    def _eval_while(self, cond, body):
        while self._eval_expr(cond):
            self._eval_statement(body)

    def _eval_print(self, expr):
        self.output.append(self._to_print(self._eval_expr(expr)))

    def _to_print(self, value):
        match value:
            case bool(b): return "true" if b else "false"
            case v if callable(v): return "<builtin>"
            case ["func", *_]: return "<func>"
            case _: return value

    def _eval_expr(self, expr):
        match expr:
            case int(value) | bool(value): return value
            case str(name): return self._eval_variable(name)
            case ["func", param, body]: return ["func", param, body, self._env]
            case ["^", a, b]: return self._eval_expr(a) ** self._eval_expr(b)
            case ["*", a, b]: return self._eval_expr(a) * self._eval_expr(b)
            case ["/", a, b]: return self._div(self._eval_expr(a), self._eval_expr(b))
            case ["+", a, b]: return self._eval_expr(a) + self._eval_expr(b)
            case ["-", a, b]: return self._eval_expr(a) - self._eval_expr(b)
            case ["=", a, b]: return self._eval_expr(a) == self._eval_expr(b)
            case ["#", a, b]: return self._eval_expr(a) != self._eval_expr(b)
            case [func, *args]:
                return self._apply(self._eval_expr(func),
                                   [self._eval_expr(arg) for arg in args])
            case unexpected: assert False, f"Internal Error at `{unexpected}`."

    def _div(self, a, b):
        assert b != 0, f"Division by zero."
        return a // b

    def _apply(self, func, args):
        if callable(func): return func(*args)

        [_, parameters, body, env] = func
        parent_env = self._env
        self._env = Environment(env)
        for param, arg in zip(parameters, args): self._env.define(param, arg)
        value = 0
        try:
            self._eval_statement(body)
        except Return as ret:
            value = ret.value
        self._env = parent_env
        return value

    def _eval_variable(self, name):
        return self._env.get(name)

if __name__ == "__main__":
    import sys

    evaluator = Evaluator()
    while True:
        print("Input source and enter Ctrl+D:")
        if (source := sys.stdin.read()) == "": break

        print("Output:")
        try:
            ast = Parser(source).parse_program()
            print(ast)
            evaluator.eval_program(ast)
            print(*evaluator.output, sep="\n")
        except AssertionError as e:
            print("Error:", e)
```