# Magento-Bug-Fixes #
Here I will provide some basic fixes for Magento Bugs and encourage good developers for better coding! Include all override and bug fixing code to fix Magento issues.

## Magento 1.14.1.0 CLI reindexall error log issue ##

Here is the error messages that you might getting when you reindex using command line in the latest Magento Enterpise 1.14.1.0

    2015-01-15T04:50:41+00:00 ERR (3): Warning: session_start(): Cannot send session cookie - headers already sent by (output started at C:\Apache2\htdocs\m1.14.1\shell\indexer.php:174)  in C:\htdocs\m1.14.1\app\code\core\Mage\Core\Model\Session\Abstract\Varien.php on line 133

    2015-01-15T04:50:41+00:00 ERR (3): Warning: session_start(): Cannot send session cache limiter - headers already sent (output started at C:\Apache2\htdocs\m1.14.1\shell\indexer.php:174)  in C:\Apache2\htdocs\m1.14.1\app\code\core\Mage\Core\Model\Session\Abstract\Varien.php on line 133

Any error messages in the /var/system.log is no good no matter it is "warning" or "error", which means something is wrong. So to fix this I just provide the quick fix without code change:

> System -> configuration -> Web -> Session Validation Settings -> Use SID on Frontend -> "No"

This will disable the SID in the frontend to not allow customer stay logged in when switching stores but this will not generate error log in the system.log

The actually course of this is class "Mage_Core_Model_Url":
    protected function _prepareSessionUrlWithParams($url, array $params)
    {
        if (!$this->getUseSession()) {
            return $this;
        }

        /** @var $session Mage_Core_Model_Session */
        $session = Mage::getSingleton('core/session', $params);

        $sessionId = $session->getSessionIdForHost($url);
        if (Mage::app()->getUseSessionVar() && !$sessionId) {
            $this->setQueryParam('___SID', $this->getSecure() ? 'S' : 'U'); // Secure/Unsecure
        } else if ($sessionId) {
            $this->setQueryParam($session->getSessionIdQueryParam(), $sessionId);
        }
        return $this;
    }

The code when initial the session in command line we already echo things and session initialization will failed as header already sent! And command line don't need to use session.

	$session = Mage::getSingleton('core/session', $params);

To fix it in code level you might try override the core session class to prevent session initialization when it is requested in shell or CLI mode:

	class Yournamespace_Core_Model_Session extends Mage_Core_Model_Session
	{
	    public function __construct($data=array())
	    {
	        if (!isset($_SERVER['REQUEST_METHOD'])) {
	            return;
	        }
	        $name = isset($data['name']) ? $data['name'] : null;
	        $this->init('core', $name);
	    }
		....
	}
