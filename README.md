# FPC and CC-tools integration

This guide shows how to deploy and run a chaincode using cc-tools on an FPC network ensuring the confidentiality requirements achieved by [fabric-private-chaincode](https://github.com/hyperledger/fabric-private-chaincode).

## MOTIVATION

In the rapidly evolving landscape of blockchain technology, privacy and security remain paramount concerns, especially in enterprise applications where sensitive data is processed. While Hyperledger Fabric provides a robust platform for developing permissioned blockchain solutions, the need for enhanced privacy in smart contract execution has led to the development of Fabric Private Chaincode (FPC). FPC ensures that both code and data are protected during execution by leveraging trusted execution environments (TEEs), thereby enabling secure and confidential transactions.

On the other hand, the complexity of developing, testing, and deploying smart contracts (chaincode) in a decentralized environment presents its own challenges. The CC Tools from Hyperledger Labs are designed to streamline this process by providing a simplified, user-friendly framework for chaincode lifecycle management. It reduces the time and effort required for developers to build and manage chaincode, facilitating quicker iterations and deployments.

This project seeks to integrate the privacy guarantees of FPC with the developer-friendly environment of CC Tools to create a unified solution that offers both enhanced security and ease of development. By doing so, it not only improves the privacy model of smart contract execution but also simplifies the workflow for developers, making the benefits of FPC more accessible. This integration serves as a powerful tool for organizations seeking to develop secure and private decentralized applications efficiently, without compromising on usability or privacy.

## PROBLEM STATEMENT

In the context of developing decentralized applications on Hyperledger Fabric, ensuring both ease of development and enhanced privacy is a critical challenge. The Fabric Private Chaincode (FPC) project provides strong privacy guarantees by enabling the execution of smart contracts within trusted execution environments (TEEs), ensuring that sensitive data and contract logic remain confidential. However, the process of developing and managing FPC chaincode remains complex, requiring significant effort in terms of lifecycle management, secure execution, and data protection.

Simultaneously, CC Tools from Hyperledger Labs simplifies the chaincode lifecycle by providing tools to automate and streamline development, testing, and deployment. However, CC Tools lacks out-of-the-box support for FPC, leading to a fragmented development workflow that doesn't take full advantage of FPC's privacy capabilities while retaining CC Tools' development simplicity.

The integration between FPC and CC Tools needs to occur at two distinct levels:

Chaincode Level: The chaincode itself must be compatible with both the privacy features of FPC and the simplified lifecycle management offered by CC Tools. This requires modifications to the chaincode lifecycle, ensuring that it can be deployed securely using FPC, while retaining the flexibility of CC Tools for development and deployment.

API Server (Client) Level: The API server (which is the client) that communicates with the peers must be adapted to handle the secure interaction between the FPC chaincode and the Fabric network. This includes managing secure communication with the TEE and ensuring that FPC’s privacy guarantees are upheld during transaction processing, while also utilizing CC Tools for chaincode operations.

Addressing these challenges is critical to enabling the seamless development and deployment of private chaincode on Hyperledger Fabric, ensuring both security and usability.


### ON THE CHAINCODE LEVEL

In developing the chaincode for the integration of FPC and CC Tools, there are specific requirements to ensure that FPC’s privacy and security features are properly leveraged. First, both FPC and CC Tools wrap the shim.ChaincodeStub interface from Hyperledger Fabric, but they must be used in a specific order to avoid conflicts. To meet this requirement, the chaincode must be wrapped with the FPC stubwrapper before being passed to the CC Tools wrapper. This ensures that FPC’s privacy protections, such as secure execution within a trusted execution environment (TEE), are properly applied before the chaincode is handled by CC Tools for lifecycle management.

![wrappingOrder](./wrappingOrder.png)

Here's an example of how we did it for the chaincode on cc-tools-demo with chaincode as a service

```
func runCCaaS() error {
	address := os.Getenv("CHAINCODE_SERVER_ADDRESS")
	ccid := os.Getenv("CHAINCODE_PKG_ID")

	tlsProps, err := getTLSProperties()
	if err != nil {
		return err
	}

	var cc shim.Chaincode

	if os.Getenv("FPC_MODE") == "true" {
        //*Wrap the chaincode with FPC wrapper*//
		cc = fpc.NewPrivateChaincode(new(CCDemo))
	} else {
		cc = new(CCDemo)
	}

	server := &shim.ChaincodeServer{
		CCID:     ccid,
		Address:  address,
		CC:       cc,
		TLSProps: *tlsProps,
	}

	return server.Start()
}
```
<br>

Secondly, the chaincode deployment process must follow the FPC deployment flow, as CC Tools adheres to the standard Fabric chaincode deployment process, which does not account for FPC’s privacy enhancements. Since FPC is already built on top of Fabric’s standard processes, ensuring that the deployment follows the FPC-specific flow will be sufficient for integrating the two tools. This means taking care to use FPC’s specialized lifecycle commands and processes during deployment, to ensure both privacy and compatibility with the broader Hyperledger Fabric framework.


![fpcFlow](./fpcFlow.png)


### ON THE API SERVER (CLIENT) LEVEL

The transaction client invocation process, as illustrated in the diagram, consists of several key stages that require careful integration between FPC and CC Tools. CC Tools supports deploying an API server over an HTTP channel, which listens for requests on a specified port. Stages 1 through 4 are managed entirely on the client side and involve handling the request, determining the appropriate transaction invocation based on the requested endpoint, and ensuring the payload is correctly parsed into a format that is FPC-friendly. This parsing step is crucial, as it prepares the data to meet FPC’s privacy and security requirements before it reaches the peer. In stage 4, the Chaincode API manages this final transformation of the request payload before passing it to the peer.

![CCAPIFlow](./CCAPIFlow.png)

At stage 5, FPC takes over, handling the transaction request on the peer side, ensuring the chaincode is executed securely within a Trusted Execution Environment (TEE). This stage ensures that both the privacy of the transaction and the integrity of the chaincode are maintained according to FPC's trusted execution protocols. More details about this stage is illustrated with the following diagram

![fpcClientFlow](./fpcClientFlow.png)

Proper handling of stages 1 to 4 is essential to guarantee that the client-side preparation is fully compatible with the FPC’s execution flow on the peer.
