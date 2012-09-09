package com.centrica.jcr.load.xml;

import java.io.File;
import java.io.FileFilter;
import java.io.FileInputStream;
import java.io.IOException;
import java.net.URISyntaxException;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;

import javax.jcr.ImportUUIDBehavior;
import javax.jcr.InvalidSerializedDataException;
import javax.jcr.ItemExistsException;
import javax.jcr.Node;
import javax.jcr.NodeIterator;
import javax.jcr.PathNotFoundException;
import javax.jcr.Property;
import javax.jcr.PropertyIterator;
import javax.jcr.Repository;
import javax.jcr.RepositoryException;
import javax.jcr.Session;
import javax.jcr.SimpleCredentials;
import javax.jcr.Value;
import javax.jcr.lock.LockException;
import javax.jcr.nodetype.ConstraintViolationException;
import javax.jcr.version.VersionException;

import org.apache.commons.lang3.StringUtils;
import org.apache.jackrabbit.commons.JcrUtils;
import org.apache.jackrabbit.rmi.repository.URLRemoteRepository;

/**
 * Imports an example XML file and outputs the contents of the entire workspace.
 */
public class XmlLoaderTryout {

	private static final String WORKSPACE = "crx.default";
	private static final String REPOSITORY_URI = "rmi://localhost:1234/crx";

	/**
	 * The main entry point of the example application.
	 * 
	 * @param args
	 *            <ol>
	 *            <li>a relative path of the node to add content under (or to
	 *            replace entirely), for example
	 *            <b>/content/sysadmin/products</b>;</li>
	 *            <li>character string, <b>replace</b>. This argument is
	 *            optional. Default is to add any values found in input, whether
	 *            they are duplicates or not.</li>
	 *            </ol>
	 * <br>
	 *            Input values are searched for - currently in XML files - in
	 *            the class path of this program. File names are expected to
	 *            start with the same character string as the last element in
	 *            the relative node path. All files that are found to match the
	 *            starting string will be loaded.<br/>
	 *            For example, when adding a node under the relative node path
	 *            <b>/content/sysadmin/products</b>, the file
	 *            <b>products_product1.xml</b> will be read, (and so will files
	 *            <b>products_product2.xml</b> and
	 *            <b>products_product2.xml</b>). <br/>
	 * <br/>
	 *            Care should be taken to avoid use of node paths and input
	 *            files with overlapping names: this may lead to this loading
	 *            utility giving unwanted results.
	 * @throws Exception
	 *             if an error occurs
	 */
	public static void main(String[] args) throws Exception {

		if (args.length <= 0) {
			throw new IllegalArgumentException("Missing program parameters.");
		}
		final String relativeNodeName = args[0].toLowerCase();

		boolean replace = false;
		if (null != args[1] && "replace".equals(args[1].toLowerCase())) {
			replace = true;
		}

		Repository repository = JcrUtils.getRepository(REPOSITORY_URI); /*
																		 * RMI
																		 * is
																		 * expected
																		 * to be
																		 * enabled
																		 */
		Session session = repository.login(new SimpleCredentials("admin",
				"admin".toCharArray()), WORKSPACE);
		String user = session.getUserID(); 
		String name = repository.getDescriptor(Repository.REP_NAME_DESC); 
		System.out.println("Logged in as " + user + " to a " + name + " repository.");

		addNodes(session, relativeNodeName, replace);

	}

	private static void addNodes(Session session, String relativeNodeName,
			boolean replace) {
		Node n = null;
		try {
			Node root = session.getRootNode();
			printChildNodeNames(root);

			if (replace) {
				if (root.hasNode(relativeNodeName)) {
					n = root.getNode(relativeNodeName);
					n.remove();
					session.save();
					System.out.println("Just called remove for: " + relativeNodeName);
					printChildNodeNames(root);
				}
			}

			n = JcrUtils
					.getOrAddNode(root, relativeNodeName, "nt:unstructured");

			String[] nodeNames = StringUtils.split(relativeNodeName, "/");

			addFileContents(session, nodeNames[nodeNames.length - 1], n);

			/*
			 * for illustration...output the repository content, underneath the
			 * newly added node(s)
			 */
			dump(n);

		} catch (RepositoryException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} finally {
			session.logout();
		}
	}

	private static void addFileContents(Session session, final String nodeName,
			Node containingNode) {

		System.out.println("Thread wise, where am i? : "
				+ Thread.currentThread().getName());
		Enumeration<URL> urls;
		try {
			/*
			 * look for matching files (at the top of the classloaders 'tree' of
			 * resources)
			 */
			urls = XmlLoaderTryout.class.getClassLoader().getResources("");

			URL url = null;
			File[] files = null;
			if (urls.hasMoreElements()) {
				url = urls.nextElement();
				File file = new File(url.toURI());

				final List<File> foundFiles = new ArrayList<File>();

				FileFilter customFilter = new FileFilter() {
					@Override
					public boolean accept(File pathname) {

						if (pathname.isDirectory()) {
							pathname.listFiles(this);
						}

						if (pathname.getName().startsWith(nodeName)) {
							foundFiles.add(pathname);
							return true;
						}
						return false;
					}

				};

				files = file.listFiles(customFilter); /*
													 * ...and process any file
													 * that matches, with
													 * reasonable accuracy, the
													 * supplied program
													 * parameter
													 */

				for (File f : files) {
					FileInputStream xml = new FileInputStream(f);
					session.importXML(containingNode.getPath(), xml,
							ImportUUIDBehavior.IMPORT_UUID_CREATE_NEW);
					xml.close();
					session.save();
					System.out.println("done for file " + f.getAbsolutePath());

				}
			}
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

	private static void printChildNodeNames(Node node)
			throws RepositoryException {
		StringBuilder sb = new StringBuilder();
		sb.append("Looping over child node paths (for " + node.getName()
				+ ")..." + "\n");
		for (Node n : JcrUtils.getChildNodes(node)) {
			sb.append(n.getName() + "\n");
		}
		System.out.println(sb);
	}

	/** Recursively outputs the contents of the given node. */
	private static void dump(Node node) throws RepositoryException {
		// First output the node path
		System.out.println(node.getPath());
		// Skip the virtual (and large!) jcr:system subtree
		if (node.getName().equals("jcr:system")) {
			return;
		}

		// Then output the properties
		PropertyIterator properties = node.getProperties();
		while (properties.hasNext()) {
			Property property = properties.nextProperty();
			if (property.getDefinition().isMultiple()) {
				// A multi-valued property, print all values
				Value[] values = property.getValues();
				for (int i = 0; i < values.length; i++) {
					System.out.println(property.getPath() + " = "
							+ values[i].getString());
				}
			} else {
				// A single-valued property
				System.out.println(property.getPath() + " = "
						+ property.getString());
			}
		}

		// Finally output all the child nodes recursively
		NodeIterator nodes = node.getNodes();
		while (nodes.hasNext()) {
			dump(nodes.nextNode());
		}
	}

}