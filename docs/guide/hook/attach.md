# Agent加载

|类全限定名称|方法|参数|备注|
|:----:|:----:|:----:|:----|
|sun.tools.attach.HotSpotVirtualMachine|loadAgentLibrary|(String agentLibrary, boolean isAbsolute, String options)| 加载 native agent（c++语言构建的agent）|
|sun.tools.attach.HotSpotVirtualMachine|loadAgent|(String agent, String options)|  加载 java agent （基于instrument api构建）|

> sun.tools.attach.HotSpotVirtualMachine

+ 加载 native agent（c++语言构建的agent）
```java
    /*
     * Load agent library
     * If isAbsolute is true then the agent library is the absolute path
     * to the library and thus will not be expanded in the target VM.
     * if isAbsolute is false then the agent library is just a library
     * name and it will be expended in the target VM.
     */
    private void loadAgentLibrary(String agentLibrary, boolean isAbsolute, String options)
        throws AgentLoadException, AgentInitializationException, IOException
    {
        InputStream in = execute("load",
                                 agentLibrary,
                                 isAbsolute ? "true" : "false",
                                 options);
        try {
            int result = readInt(in);
            if (result != 0) {
                throw new AgentInitializationException("Agent_OnAttach failed", result);
            }
        } finally {
            in.close();

        }
    }
```
+ 加载 java agent （基于instrument api构建的java agent）
```java
    /*
     * Load JPLIS agent which will load the agent JAR file and invoke
     * the agentmain method.
     */
    public void loadAgent(String agent, String options)
        throws AgentLoadException, AgentInitializationException, IOException
    {
        String args = agent;
        if (options != null) {
            args = args + "=" + options;
        }
        try {
            loadAgentLibrary("instrument", args);
        } catch (AgentLoadException x) {
            throw new InternalError("instrument library is missing in target VM", x);
        } catch (AgentInitializationException x) {
            /*
             * Translate interesting errors into the right exception and
             * message (FIXME: create a better interface to the instrument
             * implementation so this isn't necessary)
             */
            int rc = x.returnValue();
            switch (rc) {
                case JNI_ENOMEM:
                    throw new AgentLoadException("Insuffient memory");
                case ATTACH_ERROR_BADJAR:
                    throw new AgentLoadException("Agent JAR not found or no Agent-Class attribute");
                case ATTACH_ERROR_NOTONCP:
                    throw new AgentLoadException("Unable to add JAR file to system class path");
                case ATTACH_ERROR_STARTFAIL:
                    throw new AgentInitializationException("Agent JAR loaded but agent failed to initialize");
                default :
                    throw new AgentLoadException("Failed to load agent - unknown reason: " + rc);
            }
        }
    }
```
