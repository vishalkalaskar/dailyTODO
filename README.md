1.stream creation methods
Stream<Integer> stream = Stream.of(1,2,3,4,5);

int[] numbers = {1,2,3,4,5};
IntStream stream = Arrays.stream();

List<String> names = Arrays.asList("Alice","Bob","charline");
String<String> nameStream = names.stream();

2.intermediate opertions.-->return a new stream
filter,map,flatMap(List::stream),distinct(),sorted(),peek()

3.Terminal Operation-->produce a result
forEach,count(),collect(),reduce(),min(),max(),anyMatch(),allMatch(),noneMatch(),findfirst(),findAny(),

4.special stream--java provides pririmitive specific streams for efficiency.
intStream,LongStream,DoubleStream

5.paralle streams.
numbers.parallelStream().forEach(System.out::println);
