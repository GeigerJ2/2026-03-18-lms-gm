# Remove redundant `kpoints` from method names in `KpointsData`

## Problem

Every method in `KpointsData` repeats "kpoints" in its name, even though the class already provides that context:

```python
# aiida/orm/nodes/data/array/kpoints.py
class KpointsData(ArrayData):
    def set_kpoints(self, kpoints, cartesian=False, labels=None, weights=None, fill_values=0): ...
    def get_kpoints(self, also_weights=False, cartesian=False): ...
    def set_kpoints_mesh(self, mesh, offset=None): ...
    def get_kpoints_mesh(self, print_list=False): ...
    def set_kpoints_mesh_from_density(self, distance, offset=None, force_parity=False): ...
```

User code becomes redundant:

```python
kpoints = KpointsData()
kpoints.set_kpoints_mesh([4, 4, 4])
mesh = kpoints.get_kpoints_mesh()
```

## Proposed fix

Rename methods to drop the redundant prefix:

```python
class KpointsData(ArrayData):
    def set_points(self, kpoints, cartesian=False, labels=None, weights=None, fill_values=0): ...
    def get_points(self, also_weights=False, cartesian=False): ...
    def set_mesh(self, mesh, offset=None): ...
    def get_mesh(self, print_list=False): ...
    def set_mesh_from_density(self, distance, offset=None, force_parity=False): ...
```

User code becomes:

```python
kpoints = KpointsData()
kpoints.set_mesh([4, 4, 4])
mesh = kpoints.get_mesh()
```

Keep the old names as deprecated aliases during transition:

```python
def set_kpoints_mesh(self, *args, **kwargs):
    warn_deprecation('Use KpointsData.set_mesh() instead', version=3)
    return self.set_mesh(*args, **kwargs)
```

## Files affected

- `src/aiida/orm/nodes/data/array/kpoints.py`
- All call sites using `set_kpoints_mesh`, `get_kpoints_mesh`, etc.
