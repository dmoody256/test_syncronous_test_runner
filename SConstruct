import SCons

# A bound map of stream (as in stream of work) name to side-effect
# file. Since SCons will not allow tasks with a shared side-effect
# to execute concurrently, this gives us a way to limit link jobs
# independently of overall SCons concurrency.
node_map = dict()

# A list of nodes that have to be run synchronously.
# sync node ensures the test runners are syncrhonous amongst
# themselves.
sync_nodes = list()

# this emitter will make a phony sideeffect per target
# the test builders will share all the other sideeffects making
# sure the tests only run when nothing else is running.
def sync_se_emitter(target, source, env):
    name = str(target[0])
    se_name = "#unique_node_" + str(hash(name))
    se_node = node_map.get(se_name, None)
    if not se_node:
        se_node = env.Entry(se_name)
        # This may not be necessary, but why chance it
        env.NoCache(se_node)
        node_map[se_name] = se_node
        for sync_node in sync_nodes:
            env.SideEffect(se_name, sync_node)
    env.SideEffect(se_node, target)
    return (target, source)

# here we force all builders to use the emitter, so all
# targets will respect the shared sideeffect when being built.
# NOTE: that the builders which should be synchronous must be listed
# by name, as SynchronousTestRunner is in this example
original_create_nodes = SCons.Builder.BuilderBase._create_nodes
def always_emitter_create_nodes(self, env, target = None, source = None):
    if self.get_name(env) != "SynchronousTestRunner":
        if self.emitter:
            self.emitter = SCons.Builder.ListEmitter([self.emitter, sync_se_emitter])
        else:
            self.emitter = SCons.Builder.ListEmitter([sync_se_emitter])
    return original_create_nodes(self, env, target, source)
SCons.Builder.BuilderBase._create_nodes = always_emitter_create_nodes


env = Environment()
env.Tool('textfile')
nodes = []

# this is a fake test runner which acts like its running a test
env['BUILDERS']["SynchronousTestRunner"] = SCons.Builder.Builder(
    action=SCons.Action.Action([
        "sleep 1",
        "echo Starting test $TARGET",
        "sleep 5",
        "echo Finished test $TARGET",
        'echo done > $TARGET'],
    None))

# this emitter connects the test runners with the shared sideeffect
def sync_test_emitter(target, source, env):
    for name in node_map:
        env.SideEffect(name, target)
    sync_nodes.append(target)
    return (target, source)

env['BUILDERS']["SynchronousTestRunner"].emitter = SCons.Builder.ListEmitter([sync_test_emitter])

# in this test we create two test runners and make them depend on various source files
# being generated. This is just to force the tests to be run in the middle of
# the build. This will allow the example to demonstrate that all other jobs
# have paused so the test can be performed.
env.SynchronousTestRunner("test.out", "source10.c")
env.SynchronousTestRunner("test2.out", "source62.c")

for i in range(50):
    nodes.append(env.Textfile(f"source{i}.c", f"int func{i}(){{return {i};}}"))

for i in range(50, 76):
    node = env.Textfile(f"source{i}.c", f"int func{i}(){{return {i};}}")
    env.Depends(node, "test.out")
    nodes.append(node)

for i in range(76, 100):
    node = env.Textfile(f"source{i}.c", f"int func{i}(){{return {i};}}")
    env.Depends(node, "test2.out")
    nodes.append(node)
nodes.append(env.Textfile('main.c', 'int main(){return 0;}'))

env.Program('out', nodes)


