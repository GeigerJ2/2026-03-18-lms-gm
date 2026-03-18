<!-- # API: Avoid global configuration -->
<!---->
<!-- Use good defaults and let the user override them -->
<!---->
<!-- ```python -->
<!-- # Bad — mutable global that affects ALL callers -->
<!-- DEFAULT_TIMEOUT = 10 -->
<!---->
<!---->
<!-- def order(chips=None, fish=None, timeout=None): -->
<!--     if timeout is None: -->
<!--         timeout = DEFAULT_TIMEOUT  # someone else set this to 1... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- How? Module attributes are just dict entries — anyone can reassign them: -->
<!---->
<!-- ```python -->
<!-- # User thinks it's a config knob (the name invites this) -->
<!-- import fishnchips -->
<!-- fishnchips.DEFAULT_TIMEOUT = 1 -->
<!---->
<!-- # Test pollution — no teardown, leaks into subsequent tests -->
<!-- def test_quick_order(): -->
<!--     fishnchips.DEFAULT_TIMEOUT = 1  # speed up tests -->
<!--     ... -->
<!--     # forgot to reset — every test that runs after is affected -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Avoid global configuration -->
<!---->
<!-- ```python -->
<!-- # Good — keyword argument with a sensible default -->
<!-- def order(chips=None, fish=None, timeout=10): ... -->
<!---->
<!---->
<!-- # User who wants a different default? functools.partial -->
<!-- import functools -->
<!---->
<!-- order_fast = functools.partial(fishnchips.order, timeout=1) -->
<!-- ``` -->
<!---->
<!---->
<!---->
<!-- <v-click> -->
<!---->
<!-- Each caller gets exactly what they asked for. No spooky action at a distance. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Avoid global state — use a class -->
<!---->
<!-- What if `fishnchips` tracked orders in a module-level list? -->
<!---->
<!-- ```python -->
<!-- # fishnchips.py — Bad: module-level state, only one order queue -->
<!-- _orders = [] -->
<!---->
<!---->
<!-- def order(chips=0, fish=0): -->
<!--     _orders.append({"chips": chips, "fish": fish}) -->
<!---->
<!---->
<!-- # app.py — works fine... until you need TWO shops -->
<!-- fishnchips.order(chips=2, fish=1) -->
<!-- # Later: "I want separate queues for dine-in and takeaway" -->
<!-- # ...but there's only one _orders list. No way to separate them. -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Avoid global state — use a class -->
<!---->
<!-- ```python -->
<!-- # Good — state lives in a class instance -->
<!-- class Shop: -->
<!--     def __init__(self, name: str): -->
<!--         self.name = name -->
<!--         self._orders: list[dict] = [] -->
<!---->
<!--     def order(self, chips: int = 0, fish: int = 0) -> None: -->
<!--         self._orders.append({"chips": chips, "fish": fish}) -->
<!---->
<!---->
<!-- # Users can have as many shops as they want -->
<!-- dine_in = fishnchips.Shop("Dine-in") -->
<!-- takeaway = fishnchips.Shop("Takeaway") -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- When your library needs state between calls, a class is the right tool. -->
<!-- <!-- See `requests.Session` — holds auth, headers, and reuses TCP connections. --> -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Avoid global state — in aiida-core -->
<!---->
<!-- Four `global` singletons scattered across the codebase ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/manage/configuration/__init__.py -->
<!-- CONFIG: Optional['Config'] = None -->
<!---->
<!-- def get_config(create=False) -> 'Config': -->
<!--     global CONFIG  # noqa: PLW0603 -->
<!--     ... -->
<!-- ``` -->
<!---->
<!-- Also: `MANAGER`, `PROGRESS_REPORTER`, `OBJECT_LOADER` — same pattern. -->
<!---->
<!-- <v-click> -->
<!---->
<!-- The saving grace: `profile_context()` and `enable_caching()` wrap the most dangerous state in context managers. But the globals remain — not thread-safe, not async-safe. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Avoid global state — in aiida-core -->
<!---->
<!-- For `CONFIG` specifically: replace `global` with `contextvars.ContextVar` (PEP 567): -->
<!---->
<!-- ```python -->
<!-- from contextvars import ContextVar -->
<!---->
<!-- _config: ContextVar[Config | None] = ContextVar('_config', default=None) -->
<!---->
<!-- def get_config() -> Config: -->
<!--     config = _config.get()          # no global keyword needed -->
<!--     ... -->
<!---->
<!-- @contextmanager -->
<!-- def config_context(profile=None): -->
<!--     token = _config.set(load_config())  # scoped, thread/async-safe -->
<!--     try: yield -->
<!--     finally: _config.reset(token)       # clean restore via token -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Errors — raise exceptions, not error codes -->
<!---->
<!-- Build a custom exception hierarchy -->
<!---->
<!-- ```python -->
<!-- # Bad — returning None or error codes, caller has to check manually -->
<!-- def order(chips=None, fish=None): -->
<!--     response = _call_api(chips, fish) -->
<!--     if response.status_code == 200:  # Response is OK, happy path -->
<!--         return response.json() -->
<!--     return None  # caller must remember to check for None! -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Errors — raise exceptions, not error codes -->
<!---->
<!-- ```python -->
<!-- # Good — custom exception hierarchy with useful attributes -->
<!-- class Error(Exception): ... -->
<!---->
<!---->
<!-- class OrderError(Error): -->
<!--     def __init__(self, code, chef_name, message): -->
<!--         self.code = code -->
<!--         self.chef_name = chef_name -->
<!--         self.message = message -->
<!-- ``` -->
<!---->
<!-- ```python -->
<!-- # User code — easy to catch everything, or be specific -->
<!-- try: -->
<!--     fishnchips.order(chips=1, fish=2) -->
<!-- except fishnchips.OrderError as e: -->
<!--     print(f"{e.chef_name} said: {e.message}") -->
<!-- except fishnchips.Error: -->
<!--     print("Unexpected error") -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Errors — in aiida-core -->
<!---->
<!-- Semantic exception hierarchy with useful subclasses ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/common/exceptions.py -->
<!-- class MissingEntryPointError(EntryPointError): -->
<!--     """Raised when the requested entry point is not registered.""" -->
<!---->
<!-- class LockedProfileError(AiidaException): -->
<!--     """Raised if attempting to access a locked profile.""" -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- <div class="grid grid-cols-2 gap-4"> -->
<!-- <div> -->
<!---->
<!-- bare `Exception` instead of AiiDA exception ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/orm/logs.py -->
<!-- if not isinstance(entity, nodes.Node): -->
<!--     raise Exception( -->
<!--         'Only node logs are stored' -->
<!--     )  # bare Exception! -->
<!-- ``` -->
<!---->
<!-- </div> -->
<!---->
<!-- <div> -->
<!---->
<!-- overly broad `except Exception: pass` ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/transports/plugins/async_backend.py -->
<!-- try: -->
<!--     result = subprocess.run(['ssh', '-V'], ...) -->
<!--     match = re.search(r'OpenSSH_(\d+)', output) -->
<!--     ... -->
<!-- except Exception:  # silently swallows ALL errors -->
<!--     pass -->
<!-- ``` -->
<!---->
<!-- </div> -->
<!-- </div> -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <!-- # API: Backwards compatibility — keyword args are your friend --> -->
<!-- <!----> -->
<!-- <!-- How positional args break callers --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # v1.0 --> -->
<!-- <!-- def order(chips: int, fish: int) -> Receipt: ... --> -->
<!-- <!----> -->
<!-- <!-- order(2, 1)  # 2 chips, 1 fish --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- <v-click> --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # v1.1 — "fish should come first, it's fish & chips after all" --> -->
<!-- <!-- def order(fish: int, chips: int) -> Receipt: ... --> -->
<!-- <!----> -->
<!-- <!-- order(2, 1)  # now 2 fish, 1 chip — silently wrong! --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- </v-click> --> -->
<!-- <!----> -->
<!-- <!-- --- --> -->
<!-- <!----> -->
<!-- <!-- # API: Backwards compatibility — keyword args are your friend --> -->
<!-- <!----> -->
<!-- <!-- Add features without breaking existing callers --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # v1.0 — simple --> -->
<!-- <!-- def order(chips: int = 0, fish: int = 0) -> Receipt: ... --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- <v-click> --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # v1.1 — added fish_type, fully backwards compatible --> -->
<!-- <!-- from enum import Enum --> -->
<!-- <!----> -->
<!-- <!----> -->
<!-- <!-- class FishType(Enum): --> -->
<!-- <!--     BATTERED = "battered" --> -->
<!-- <!--     CRUMBED = "crumbed" --> -->
<!-- <!----> -->
<!-- <!----> -->
<!-- <!-- def order( --> -->
<!-- <!--     chips: int = 0, fish: int = 0, fish_type: FishType = FishType.BATTERED --> -->
<!-- <!-- ) -> Receipt: ... --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- </v-click> --> -->
<!-- <!----> -->
<!-- <!-- --- --> -->
<!-- <!----> -->
<!-- <!-- # API: Backwards compatibility — keyword args are your friend --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # v1.2 — added new foods, still backwards compatible --> -->
<!-- <!-- def order( --> -->
<!-- <!--     chips: int = 0, --> -->
<!-- <!--     fish: int = 0, --> -->
<!-- <!--     fish_type: FishType = FishType.BATTERED, --> -->
<!-- <!--     onion_rings: int = 0, --> -->
<!-- <!--     hotdogs: int = 0, --> -->
<!-- <!-- ) -> Receipt: ... --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- Every old call still works. New keyword args with defaults = **no breaking changes**. --> -->
<!-- <!----> -->
<!-- <!-- --- --> -->
<!-- <!----> -->
<!-- <!-- # API: Backwards compatibility — in aiida-core --> -->
<!-- <!----> -->
<!-- <!-- Good: `Process.__init__` — all parameters have defaults --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # aiida/engine/processes/process.py --> -->
<!-- <!-- def __init__( --> -->
<!-- <!--     self, --> -->
<!-- <!--     inputs: Optional[Dict[str, Any]] = None, --> -->
<!-- <!--     logger: Optional[logging.Logger] = None, --> -->
<!-- <!--     runner: Optional['Runner'] = None, --> -->
<!-- <!--     parent_pid: Optional[int] = None, --> -->
<!-- <!--     enable_persistence: bool = True, --> -->
<!-- <!-- ) -> None: ... --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- <v-click> --> -->
<!-- <!----> -->
<!-- <!-- Bad: internal helper with 7 positional parameters --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # aiida/engine/daemon/execmanager.py --> -->
<!-- <!-- async def _copy_remote_files( --> -->
<!-- <!--     logger, node, computer, transport, --> -->
<!-- <!--     remote_copy_list, remote_symlink_list, workdir --> -->
<!-- <!-- ):  # 7 positional args — adding one breaks all callers --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- Should use keyword-only arguments: `async def _copy_remote_files(*, logger, node, ...)`. Even for internal functions, keyword args prevent ordering mistakes. --> -->
<!-- <!----> -->
<!-- <!-- </v-click> --> -->
<!-- <!----> -->
<!---->
<!-- --- -->


<!-- # API: Context managers as API -->
<!---->
<!-- Resource management the Pythonic way -->
<!---->
<!-- <div class="grid grid-cols-2 gap-4"> -->
<!-- <div> -->
<!---->
<!-- ```python -->
<!-- # Bad — easy to forget cleanup -->
<!-- conn = db.connect() -->
<!-- try: -->
<!--     result = conn.execute("SELECT ...") -->
<!-- finally: -->
<!--     conn.close()  # what if you forget? -->
<!-- ``` -->
<!---->
<!-- </div> -->
<!-- <div> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # Good — cleanup is automatic -->
<!-- with db.connect() as conn: -->
<!--     result = conn.execute("SELECT ...") -->
<!-- # conn.close() called automatically -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- </div> -->
<!-- </div> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # Typed context manager -->
<!-- from contextlib import contextmanager -->
<!-- from typing import Iterator -->
<!---->
<!-- @contextmanager -->
<!-- def connect(url: str) -> Iterator[Connection]: -->
<!--     conn = Connection(url) -->
<!--     try: -->
<!--         yield conn -->
<!--     finally: -->
<!--         conn.close() -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- If your API involves **acquire/release**, **open/close**, or **setup/teardown**, make it a context manager. Users get correctness for free. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Context managers — in aiida-core -->
<!---->
<!-- `Transport` supports nested context managers with reference counting ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/transports/transport.py -->
<!-- def __enter__(self) -> Self: -->
<!--     if self._track_enter():  # reference counting for nested use -->
<!--         self.open() -->
<!--     return self -->
<!---->
<!-- def __exit__(self, type_, value, traceback): -->
<!--     if self._track_exit(): -->
<!--         self.close() -->
<!---->
<!-- # Also supports async: -->
<!-- async def __aenter__(self): ... -->
<!-- async def __aexit__(self, ...): ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- Users can nest `with transport:` blocks safely, and both sync and async paths are supported. The transport connection is never leaked, even on exceptions. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Context managers — in aiida-core (cont.) -->
<!---->
<!-- `get_container()` returns an unmanaged resource ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/storage/psql_dos/migrator.py (before fix) -->
<!-- def get_container(self) -> Container: -->
<!--     return Container(get_filepath_container(self.profile)) -->
<!---->
<!-- # Callers forgot to close: -->
<!-- container = self.get_container() -->
<!-- container.init_container(clear=True) -->
<!-- # container is never closed → file descriptors leak -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->
<!-- # API: Context managers — in aiida-core (cont.) -->
<!---->
<!-- Fix: replace with a context manager, deprecate the old method ([#6741](https://github.com/aiidateam/aiida-core/pull/6741)) ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/storage/psql_dos/migrator.py (after fix) -->
<!-- def get_container(self): -->
<!--     from aiida.common.warnings import warn_deprecation -->
<!--     warn_deprecation( -->
<!--         'get_container() may leave resources open, use container_context() instead.', -->
<!--         version=3, -->
<!--     ) -->
<!--     return Container(get_filepath_container(self.profile)) -->
<!---->
<!-- @contextmanager -->
<!-- def container_context(self) -> Generator[Container, None, None]: -->
<!--     with Container(get_filepath_container(self.profile)) as container: -->
<!--         yield container  # Container.__exit__ handles close() -->
<!---->
<!-- # Callers now can't forget: -->
<!-- with self.container_context() as container: -->
<!--     container.init_container(clear=True) -->
<!-- # container is closed automatically -->
<!-- ``` -->
<!---->
<!-- --- -->
<!---->

<!-- # API: Don't overuse Python's expressiveness -->
<!---->
<!-- Just because you *can* doesn't mean you *should* -->
<!---->
<!-- <div class="grid grid-cols-2 gap-4"> -->
<!-- <div> -->
<!---->
<!-- ```python -->
<!-- # Bad — operator overloading on a non-numeric type -->
<!-- class Order: -->
<!--     # an Order? a list[Order]? mutates self? -->
<!--     def __add__(self, other): -->
<!--         return merge_orders(self, other) -->
<!---->
<!--     # a new order? three copies? quantities tripled? -->
<!--     def __mul__(self, n): -->
<!--         return scale_order(self, n) -->
<!-- ``` -->
<!---->
<!-- </div> -->
<!-- <div> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- # Good — explicit methods that say what they do -->
<!-- class Order: -->
<!--     def merge_with(self, other: "Order") -> "Order": ... -->
<!--     def scale(self, factor: int) -> "Order": ... -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- </div> -->
<!-- </div> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- **Rules of thumb:** -->
<!---->
<!-- - Only override `+`, `*` etc. for number/vector types -->
<!-- - Only override `[]` (`__getitem__`) for actual collections — it signals *"this is a sequence or mapping"* -->
<!-- - Property getters should be cheap — no I/O, no exceptions -->
<!-- - **If the type signature is too hard to write, the API might be a bad idea** -->
<!---->
<!-- </v-click> -->

<!-- --- -->
<!---->
<!-- # Types: NewType — preventing value mix-ups -->
<!---->
<!-- Semantically different values deserve different types -->
<!---->
<!-- ```python -->
<!-- from typing import NewType -->
<!---->
<!-- UserId = NewType("UserId", int) -->
<!-- OrderId = NewType("OrderId", int) -->
<!---->
<!---->
<!-- def get_order(order_id: OrderId) -> Order: ... -->
<!---->
<!---->
<!-- user_id = UserId(42) -->
<!-- get_order(user_id)  # error: UserId is not compatible with OrderId -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - **Zero runtime cost** — `NewType` is erased at runtime -->
<!-- - Prevents an entire class of "wrong ID" bugs -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # Types: `@final` and `Final` — locking things down -->
<!---->
<!-- Prevent accidental overrides and mutations -->
<!---->
<!-- ```python -->
<!-- from typing import final, Final -->
<!---->
<!-- MAX_RETRIES: Final = 3 -->
<!-- MAX_RETRIES = 5  # error: cannot assign to Final variable -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- from typing import final -->
<!---->
<!---->
<!-- class BaseProcessor: -->
<!--     @final -->
<!--     def validate(self, data: bytes) -> bool: -->
<!--         """Critical validation — subclasses must not override.""" -->
<!--         return len(data) > 0 and data[0:4] == b"MAGIC" -->
<!---->
<!-- class CustomProcessor(BaseProcessor): -->
<!--     def validate(self, data: bytes) -> bool:  # error: cannot override final -->
<!--         return True  # oops — security bypass prevented -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- <v-click> -->
<!---->
<!-- - `Final` for constants — prevents reassignment -->
<!-- - `@final` on methods — prevents override in subclasses -->
<!-- - `@final` on classes — prevents subclassing entirely -->
<!-- - Communicates **intent**: "this is not meant to be extended" -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- # Types: `@final` and `Final` — in aiida-core -->
<!---->
<!-- `@final` prevents subclassing config singletons ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/manage/configuration/settings.py -->
<!-- from typing import final -->
<!---->
<!-- @final -->
<!-- class AiiDAConfigDir: -->
<!--     """Singleton for setting and getting the path to configuration directory.""" -->
<!--     ... -->
<!---->
<!-- @final -->
<!-- class AiiDAConfigPathResolver: -->
<!--     """For resolving configuration directory, daemon dir, daemon log dir.""" -->
<!--     ... -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- 9 constants in the same file lack `Final` ❌ -->
<!---->
<!-- ```python -->
<!-- # aiida/manage/configuration/settings.py -->
<!-- DEFAULT_UMASK = 0o0077                       # should be Final -->
<!-- DEFAULT_AIIDA_PATH_VARIABLE = 'AIIDA_PATH'   # should be Final -->
<!-- DEFAULT_CONFIG_DIR_NAME = '.aiida'            # should be Final -->
<!-- DEFAULT_CONFIG_FILE_NAME = 'config.json'      # should be Final -->
<!-- DEFAULT_CONFIG_INDENT_SIZE = 4                # should be Final -->
<!-- # ... 4 more -->
<!-- ``` -->
<!---->
<!-- Nothing prevents `settings.DEFAULT_UMASK = 0o0000` — a subtle security issue that `Final` would catch. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

<!-- # Types: `ParamSpec` — preserve decorator signatures -->
<!---->
<!-- Don't let decorators erase your function types -->
<!---->
<!-- ```python -->
<!-- from typing import ParamSpec, TypeVar, Callable -->
<!---->
<!-- P = ParamSpec("P") -->
<!-- R = TypeVar("R") -->
<!---->
<!---->
<!-- def log(func: Callable[P, R]) -> Callable[P, R]: -->
<!--     def wrapper(*args: P.args, **kwargs: P.kwargs) -> R: -->
<!--         print(f"calling {func.__name__}") -->
<!--         return func(*args, **kwargs) -->
<!--     return wrapper -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- @log -->
<!-- def fetch(url: str, timeout: float = 10.0) -> bytes: ... -->
<!---->
<!-- fetch(42)        # error: int is not str — signature preserved! -->
<!-- fetch("https://example.com", timeout="slow")  # error: str is not float -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->
<!-- <!-- # Types: `ParamSpec` — in aiida-core --> -->
<!-- <!----> -->
<!-- <!-- `@calcfunction` preserves signatures via `ParamSpec` + Protocol ✅ --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # aiida/engine/processes/functions.py --> -->
<!-- <!-- P = ParamSpec('P') --> -->
<!-- <!-- R_co = TypeVar('R_co', covariant=True) --> -->
<!-- <!----> -->
<!-- <!-- class ProcessFunctionType(Protocol, Generic[P, R_co, N]): --> -->
<!-- <!--     def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ... --> -->
<!-- <!--     def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[...]: ... --> -->
<!-- <!--     process_class: Type[Process] --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- <v-click> --> -->
<!-- <!----> -->
<!-- <!-- the internal decorator erases types with `*args, **kwargs` ❌ --> -->
<!-- <!----> -->
<!-- <!-- ```python --> -->
<!-- <!-- # Same file, the actual decorator implementation --> -->
<!-- <!-- @functools.wraps(function) --> -->
<!-- <!-- def decorated_function(*args, **kwargs):  # signature erased! --> -->
<!-- <!--     result, _ = run_get_node(*args, **kwargs) --> -->
<!-- <!--     return result --> -->
<!-- <!----> -->
<!-- <!-- decorated_function.run = decorated_function  # type: ignore[attr-defined] --> -->
<!-- <!-- decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined] --> -->
<!-- <!-- # ... 6 more type: ignore[attr-defined] lines --> -->
<!-- <!-- ``` --> -->
<!-- <!----> -->
<!-- <!-- The `ProcessFunctionType` Protocol patches over this at the type level — but the runtime decorator still erases signatures. --> -->
<!-- <!----> -->
<!-- <!-- </v-click> --> -->
<!-- <!----> -->
<!-- <!-- --- --> -->
<!-- <!----> -->
<!---->

---

<!-- # Types: `ParamSpec` — in aiida-core -->
<!---->
<!-- `@calcfunction` preserves signatures via `ParamSpec` + Protocol ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/processes/functions.py -->
<!-- P = ParamSpec('P') -->
<!-- R_co = TypeVar('R_co', covariant=True) -->
<!---->
<!-- class ProcessFunctionType(Protocol, Generic[P, R_co, N]): -->
<!--     def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ... -->
<!--     def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[...]: ... -->
<!--     process_class: Type[Process] -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- the internal decorator erases types with `*args, **kwargs` ❌ -->
<!---->
<!-- ```python -->
<!-- # Same file, the actual decorator implementation -->
<!-- @functools.wraps(function) -->
<!-- def decorated_function(*args, **kwargs):  # signature erased! -->
<!--     result, _ = run_get_node(*args, **kwargs) -->
<!--     return result -->
<!---->
<!-- decorated_function.run = decorated_function  # type: ignore[attr-defined] -->
<!-- decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined] -->
<!-- # ... 6 more type: ignore[attr-defined] lines -->
<!-- ``` -->
<!---->
<!-- The `ProcessFunctionType` Protocol patches over this at the type level — but the runtime decorator still erases signatures. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->

<!-- # Types: `ParamSpec` — preserve decorator signatures -->
<!---->
<!-- Don't let decorators erase your function types -->
<!---->
<!-- ```python -->
<!-- from typing import ParamSpec, TypeVar, Callable -->
<!---->
<!-- P = ParamSpec("P") -->
<!-- R = TypeVar("R") -->
<!---->
<!---->
<!-- def log(func: Callable[P, R]) -> Callable[P, R]: -->
<!--     def wrapper(*args: P.args, **kwargs: P.kwargs) -> R: -->
<!--         print(f"calling {func.__name__}") -->
<!--         return func(*args, **kwargs) -->
<!--     return wrapper -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- ```python -->
<!-- @log -->
<!-- def fetch(url: str, timeout: float = 10.0) -> bytes: ... -->
<!---->
<!-- fetch(42)        # error: int is not str — signature preserved! -->
<!-- fetch("https://example.com", timeout="slow")  # error: str is not float -->
<!-- ``` -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->

<!-- # Types: `ParamSpec` — in aiida-core -->
<!---->
<!-- `@calcfunction` preserves signatures via `ParamSpec` + Protocol ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/engine/processes/functions.py -->
<!-- P = ParamSpec('P') -->
<!-- R_co = TypeVar('R_co', covariant=True) -->
<!---->
<!-- class ProcessFunctionType(Protocol, Generic[P, R_co, N]): -->
<!--     def __call__(self, *args: P.args, **kwargs: P.kwargs) -> R_co: ... -->
<!--     def run_get_node(self, *args: P.args, **kwargs: P.kwargs) -> tuple[...]: ... -->
<!--     process_class: Type[Process] -->
<!-- ``` -->
<!---->
<!-- <v-click> -->
<!---->
<!-- the internal decorator erases types with `*args, **kwargs` ❌ -->
<!---->
<!-- ```python -->
<!-- # Same file, the actual decorator implementation -->
<!-- @functools.wraps(function) -->
<!-- def decorated_function(*args, **kwargs):  # signature erased! -->
<!--     result, _ = run_get_node(*args, **kwargs) -->
<!--     return result -->
<!---->
<!-- decorated_function.run = decorated_function  # type: ignore[attr-defined] -->
<!-- decorated_function.run_get_pk = run_get_pk   # type: ignore[attr-defined] -->
<!-- # ... 6 more type: ignore[attr-defined] lines -->
<!-- ``` -->
<!---->
<!-- The `ProcessFunctionType` Protocol patches over this at the type level — but the runtime decorator still erases signatures. -->
<!---->
<!-- </v-click> -->
<!---->
<!-- --- -->
<!---->

<!-- # Types: Generics — in aiida-core -->
<!---->
<!-- full generic entity hierarchy preserves types across layers ✅ -->
<!---->
<!-- ```python -->
<!-- # aiida/orm/implementation/entities.py -->
<!-- EntityType = TypeVar('EntityType', bound='BackendEntity') -->
<!---->
<!-- class BackendCollection(Generic[EntityType]): -->
<!--     ENTITY_CLASS: ClassVar[Type[EntityType]] -->
<!---->
<!--     def create(self, **kwargs: Any) -> EntityType:  # precise return type -->
<!--         return self.ENTITY_CLASS(backend=self._backend, **kwargs) -->
<!-- ``` -->
<!---->
<!-- --- -->

