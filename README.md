# testGithub
	package com.asiainfo.template.utils;

import java.beans.BeanInfo;
import java.beans.Introspector;
import java.beans.PropertyDescriptor;
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.Reader;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.alibaba.fastjson.JSONObject;
import com.fasterxml.jackson.annotation.JsonProperty;

public class DataUtils {
	/**
	 * 验证输入参数
	 * @param input
	 * @param allChecked 
	 * @param keys
	 * @throws Exception
	 */
	public static void checkEmptyInput(Map<String, Object> input,
			boolean allChecked, String[]... keys) throws Exception {
		if (input == null || input.size() == 0)
			return;
		boolean isEmpty = false;
		String value="";
		for (int i = 0; i < keys.length; i++) {
			String[] key = keys[i];
			for (int j = 0; j < key.length; j++) {
				if (input.get(key[j]) == null || input.get(key[j]).equals("")) {
					isEmpty = true;
					value =value + "|" + key[j];
					break;
				}
				else
				{
					isEmpty = false;
				}
			}
			if((allChecked && isEmpty)||(!allChecked && 
					!isEmpty))
				break;
		}
		if(isEmpty)
			throw new BusiException("9999","["+value+"]"+ " is empty!");
	}
	
	public static void checkEmptyInput(JSONObject jsonObject,
			boolean allChecked, String[]... keys) throws Exception {		
		if (jsonObject == null || jsonObject.size() == 0)
			return;
		boolean isEmpty = false;
		String value="";
		for (int i = 0; i < keys.length; i++) {
			String[] key = keys[i];
			for (int j = 0; j < key.length; j++) {
				if (!jsonObject.containsKey(key[j])
						|| jsonObject.get(key[j]) == null
						|| jsonObject.get(key[j]).toString().equals("")) {
					isEmpty = true;
					value =value + "|" + key[j];
					break;
				}
				else
				{
					isEmpty = false;
				}
			}
			if((allChecked && isEmpty)||(!allChecked && 
					!isEmpty))
				break;
		}
		if(isEmpty)
			throw new BusiException("9999","["+value+"]"+ " is empty!");
		
	}
	
	
/**
 * map转成javabean ,支持String 转 long,string,int,java.util.Date
 * @param map
 * @param clz
 * @param underline
 * @param simpleDateFormat
 * @return
 * @throws Exception
 */

	public static <T> T map2bean(Map<String, Object> map, Class<T> clz,
			boolean underlineToCamel, SimpleDateFormat simpleDateFormat)
			throws Exception {
		HashMap<String, Object> hm = new HashMap<String, Object>();
		if (underlineToCamel) {
			for (String key : map.keySet()) {
				hm.put(underlineToCamel(key), map.get(key));
			}
		} else {
			hm.putAll(map);
		}
		// 创建JavaBean对象
		T obj = clz.newInstance();
		// 获取指定类的BeanInfo对象
		BeanInfo beanInfo = Introspector.getBeanInfo(clz, Object.class);
		// 获取所有的属性描述器
		PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
		for (PropertyDescriptor pd : pds) {
			Object value = hm.get(pd.getName());
			if (value == null)
				continue;
			Method setter = pd.getWriteMethod();
			if (setter == null)
				continue;
			if (value instanceof java.lang.String) {
				if (pd.getPropertyType().equals(java.lang.Integer.class)) {
					setter.invoke(obj,(Integer) Integer.parseInt((String) value));
				} else if (pd.getPropertyType().equals(java.lang.Long.class)) {
					setter.invoke(obj, (Long) Long.parseLong((String) value));
				} else if (pd.getPropertyType().equals(java.util.Date.class)) {
					setter.invoke(obj, DateUtils.str2Date((String) value,simpleDateFormat));
				}
				else
					setter.invoke(obj, value);
			} else {
				setter.invoke(obj, value);
			}
		}
		return obj;
	}
/**
 * javabean转成map
 * @param obj
 * @param camelToUnderline
 * @param toUpperCase
 * @return
 * @throws Exception
 */
	
	public static Map<String, Object> bean2map(Object obj,boolean camelToUnderline,boolean toUpperCase) throws Exception {
		HashMap<String, Object> hm = new HashMap<String, Object>();
		
		// 获取指定类的BeanInfo对象
		BeanInfo beanInfo = Introspector.getBeanInfo(obj.getClass(),
				Object.class);
		// 获取所有的属性描述器
		PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
		for (PropertyDescriptor pd : pds) {
			Method getter = pd.getReadMethod();
			if(getter==null)
				continue;
			String key = pd.getName();
			Object value = getter.invoke(obj);
			if(value==null)
				continue;
			if(camelToUnderline)
			{
				key = underlineToCamel(key);
			}
			if(toUpperCase)
			{
				key = key.toUpperCase();
			}
			hm.put(key, value);
		}
		return hm;
	}
/**
 *  将es的bean按照注解字段转成map
 * @param obj
 * @return
 * @throws Exception
 */
	public static Map<String, Object> esBean2Map(Object obj) throws Exception {
		HashMap<String, Object> hm = new HashMap<String, Object>();
		HashMap<String, String> keyMap = new HashMap<String, String>();

		Field[] fields = obj.getClass().getDeclaredFields();
		for (Field field : fields) {
			JsonProperty fieldAnno = field.getAnnotation(JsonProperty.class);
			if (fieldAnno == null)
			keyMap.put(field.getName(), field.getName());
			else
			keyMap.put(field.getName(), fieldAnno.value());
		}
		// 获取指定类的BeanInfo对象
		BeanInfo beanInfo = Introspector.getBeanInfo(obj.getClass(),
				Object.class);
		// 获取所有的属性描述器
		PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
		for (PropertyDescriptor pd : pds) {
			Method getter = pd.getReadMethod();
			if (getter == null)
				continue;
			String key = pd.getName();
			if (keyMap.containsKey(key) && keyMap.get(key)!=null  &&!"".equals(keyMap.get(key))) 
			{
				Object value = getter.invoke(obj);
				if(value==null)
					continue;
					hm.put(keyMap.get(key), value);
			}
		}
		return hm;
	}
	
	/**
	 * 
	 *  按照bean的注解将map转换key值
	 * @param param
	 * @param args
	 * @return
	 */
	public static <T> Map<String, Object> transEsMap(Map<String, Object> param,
			Class<T> clz, String[] args) {
		Map<String, Object> paramMap = new HashMap<String, Object>();
		Map<String, Object> resultMap = new HashMap<String, Object>();
		if (args == null) {
			paramMap.putAll(param);
		} else {
			for (String key : args) {
				if (param.containsKey(key) && param.get(key) != null)
					paramMap.put(key, param.get(key));
			}
		}
		Field[] fields = clz.getDeclaredFields();
		for (Field field : fields) {
			JsonProperty fieldAnno = field.getAnnotation(JsonProperty.class);
			if (!paramMap.containsKey(field.getName()) ||paramMap.get(field.getName())==null ||"".equals(paramMap.get(field.getName())))
				continue;
			if (fieldAnno == null)
				resultMap.put(field.getName(), paramMap.get(field.getName()));
			else
				resultMap.put(fieldAnno.value(), paramMap.get(field.getName()));
		}
		return resultMap;
	}
	/**
	 * 按照bean的注解将map转换key值
	 * @param param
	 * @param clz
	 * @return
	 */
	public static <T> Map<String, Object> transEsMap(Map<String, Object> param,Class<T> clz) {
		return transEsMap(param,clz,null);
	}
	
	public static <T> String transEsKey(String key, Class<T> clz) {
		try {
			Field field = clz.getDeclaredField(key);
			JsonProperty fieldAnno = field.getAnnotation(JsonProperty.class);
			key = fieldAnno.value();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return key;
	}
	
	/**
	 * 转换mapkey值的格式
	 * @param input
	 * @param transtype
	 * @return
	 */
	
	@SuppressWarnings({ "unchecked", "rawtypes" })
	public static Map<String, Object> mapFormat(Map<String, Object> input,
			Transtype transtype) {
		if (input == null)
			return null;
		HashMap<String, Object> hm = new HashMap<String, Object>();
		for (Map.Entry<String, Object> entry : input.entrySet()) {
			String mapKey = entry.getKey();
			Object mapValue = entry.getValue();
			if (mapValue != null) {
				if (mapValue instanceof List) {
					ArrayList<Object>arrayList = new ArrayList<Object>();
					for (int i = 0; i < ((List) mapValue).size(); i++) {
						Object obj = ((List) mapValue).get(i);
						if (obj instanceof Map) {
							obj = mapFormat((Map<String, Object>) obj, transtype);
						}
						arrayList.add(obj);
					}	
					mapValue = arrayList;
				} else if (mapValue instanceof Map) {
					mapValue = mapFormat((Map<String, Object>) mapValue, transtype);
				}
			}
			switch (transtype) {
			case camelToUnderline:
				mapKey = camelToUnderline(mapKey).toUpperCase();
				break;
			case underlineToCamel:
				mapKey = underlineToCamel(mapKey.toLowerCase());
				break;
			}
			hm.put(mapKey, mapValue);
		}
		
		return hm;
	}

	/**
	 * 转成驼峰格式
	 * @param param
	 * @return
	 */
    public static String underlineToCamel(String param){    
        if (param==null||"".equals(param.trim())){    
            return  param;    
        } 
        param = param.toLowerCase();
        int len=param.length();    
        StringBuilder sb=new StringBuilder(len);    
        for (int i = 0; i < len; i++) {    
            char c=param.charAt(i);    
            if (c=='_'){    
                if (++i<len){    
                    sb.append(Character.toUpperCase(param.charAt(i)));    
                }    
            }else{    
                sb.append(c);    
            }    
        }    
        return sb.toString();    
    }
	/**
	 * 转成下划线格式
	 * @param param
	 * @return
	 */
    public static String camelToUnderline(String param){ 
        if (param==null||"".equals(param.trim())){    
            return  param;    
        } 
        int len=param.length();    
        StringBuilder sb=new StringBuilder(len);      	
        for (int i = 0; i < len; i++) {    
            char c=param.charAt(i);    
            if (Character.isUpperCase(c)){    
            	sb.append('_');
                sb.append(Character.toLowerCase(c));    
            }else{    
                sb.append(c);    
            }    
        }
        return sb.toString();
    }

	
	public enum Transtype {
		camelToUnderline, underlineToCamel
	}
	
	public static void cpFile(File src,File target)  throws IOException {
		BufferedReader reader = null;
		BufferedWriter writer = null;
		try {
//			File src = new File(
//					"D:\\资源管理项目\\CODE\\templateParent\\templateInteApi\\src\\main\\java\\com\\asiainfo\\template\\pojo\\es\\PurchaseEsPo.java");
//			File target = new File(
//					"D:\\资源管理项目\\CODE\\templateParent\\templateInteApi\\src\\main\\java\\com\\asiainfo\\template\\pojo\\es\\PurchaseEsPo_1.java");
			reader = new BufferedReader(new InputStreamReader(new FileInputStream(src)));
			writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(target)));
			String text;
			while ((text = reader.readLine()) != null) {
				if (text.indexOf("@JsonProperty") != -1) {
					String col = text.substring(text.indexOf("@JsonProperty")
							+ "@JsonProperty".length() + 2,
							text.indexOf(")") - 1);
					text = text.replace(col, col.toUpperCase());
				}
				writer.write(text);
				writer.newLine();
				writer.flush();
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				if (writer != null) {
					writer.close();
				}
				if (reader != null) {
					reader.close();
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}



}

