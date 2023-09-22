```java
 public static void main(String[] args) throws IOException {
        File file = new File("./testExport.xlsx");
//        file.createNewFile();
//        export(new FileOutputStream(file), data, User.class, true);

        final List<User> read = read(new FileInputStream(file), User.class, Locale.CHINA);
        System.out.println(read);
    }

    @Data
    @Accessor(chain=true)
    public static class User{
        @ExcelProperty(value = "姓名")
        private String names;
        @ExcelProperty(value = "性别")
        private Integer sexs;
    }
```

该场景下无法正常获取到最后的User对象，跟踪源码至

ModelBuildEventLisenter

```java
for (Map.Entry<Integer, Head> entry : headMap.entrySet()) {
            Integer index = entry.getKey();
            if (!cellDataMap.containsKey(index)) {
                continue;
            }
            CellData cellData = cellDataMap.get(index);
            if (cellData.getType() == CellDataTypeEnum.EMPTY) {
                continue;
            }
            ExcelContentProperty excelContentProperty = contentPropertyMap.get(index);
            Object value = ConverterUtils.convertToJavaObject(cellData, excelContentProperty.getField(),
                excelContentProperty, currentReadHolder.converterMap(), currentReadHolder.globalConfiguration(),
                context.readRowHolder().getRowIndex(), index);
            if (value != null) {
                map.put(excelContentProperty.getField().getName(), value);
            }
        }
        //此处发现map中值正确，resultModel也是正确的
        BeanMap.create(resultModel).putAll(map);
        //但是在上一行代码执行后依旧没有正常数据导入
        return resultModel;
```

查看putAll的源码如下

度了一下是使用了Introspector的类的某个方法

```java
# Introspector.java 第520行
if (int.class.equals(argTypes[0]) && name.startsWith(GET_PREFIX)) {
   pd = new IndexedPropertyDescriptor(this.beanClass, name.substring(3), null, null, method, null);
   //下面这行判断，只获取返回值是void类型的setxxxx方法
 } else if (void.class.equals(resultType) && name.startsWith(SET_PREFIX)) {
    // Simple setter
    pd = new PropertyDescriptor(this.beanClass, name.substring(3), null, method);
    if (throwsException(method, PropertyVetoException.class)) {
       pd.setConstrained(true);
    }
}
```

